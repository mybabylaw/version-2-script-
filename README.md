let
    GMS_STATIONS_DATA = #"GMS Stations Data",
    GMS_LINES_DATA = #"GMS Lines data",
    TIESTATIONS_DATA = #"Tiestations data",
    JURISDICTION_BY_GRID = #"Jurisdiction By Grid",
    TOMS_SUB_QUERY = #"TOMS Sub Query",
    MAXIMO_SUB_QUERY = #"Maximo SubQuery"
 
in
    MAXIMO_SUB_QUERY




# version-2-script-

 Image  
 
import pyodbc
import pandas as pd
from collections import deque
from AllOutagesQuery import outagesQuery

# CONNECTION STRINGS
TOMS_CONN_STR = 'DRIVER={SQL Server};SERVER=LITDBETST079\TVNT150001;DATABASE=IPSEnergy193_Test;UID=CCOS_usr;PWD=HV<.D_e5nlBN'
MAXIMO_CONN_STR = 'Persist Security Info=True;DSN=GMDLMVPPBI2TDVPROD;'

# INPUT FILES
GMS_STATIONS_DATA = './Input_Files/rtnet_bus_snap_report.csv'
GMS_LINES_DATA = './Input_Files/rtnet_daily_bus_report.csv'
TIESTATIONS_DATA = './Input_Files/tiestations.xlsx'

# OUTPUT FILES 
REPORT_DATA = None

# GENERAL SETTINGS
STORM_NAME = 'Hurricane Michael'
PRUNE_ISLANDS = True
ONLY_LINES_OUTAGES = True
REPORT_TOMS_STATIONS = False
REPORT_NONENERGIZED_ISLANDS = False

# Other Data
JURISDICTION_BY_GRID = {
   'ARKANSAS': 'EAL',
   'MISSISSIPPI': 'EML',
   'GULF STATES TX': 'ETI',
   'TEXAS': 'ETI',
   'GULF STATES LA': 'ELA',
   'LOUISIANA': 'ELA',
   'NEW ORLEANS': 'ENOL',
   'GULF STATES SOUTH LA': 'EGSL'
}

TOMS_SUB_QUERY = """
   SELECT 
       MRID,
       --JSON_VALUE('["' + REPLACE(CimPathName, ' ', '","') + '"]', '$[2]') AS CimPathName
       Name AS CimPathName
   FROM ExCimElement
   WHERE CimClass = '358';
"""

MAXIMO_SUB_QUERY = """
   SELECT
       v_Locations_bc.Location AS "Name",
       v_Locations_bc.Description,
       v_Locations_bc."Type",
       v_Locations_bc.EtrGrid,
       UPPER(v_Locations_bc.Etrmrid) AS "MRID",
       v_Locations_bc.EtrMaintenanceAreaCode AS "EtrMaintenanceArea",
       v_Locations_bc.ETRMRIDNAME,
       v_Locations_bc.EtrSubstnCompany,
       COALESCE(v_Locations_bc.ETRSYSTEMKV, v_Locations_bc.ETRVOLTAGE1) AS "EtrVoltage1",
       v_Locations_bc.Etrlatitude AS "Latitudey",
       v_Locations_bc.Etrlongitude AS "Longitudex",
       v_ServiceAddress_bc.County
   FROM GMDL.Maximo_Su.v_Locations_bc
   LEFT JOIN GMDL.Maximo_Su.v_ServiceAddress_bc on v_Locations_bc.saddresscode = v_ServiceAddress_bc.addresscode and v_Locations_bc.orgid = v_ServiceAddress_bc.orgid
   WHERE v_Locations_bc."Type" IN ('SUBSTATION')
       AND v_Locations_bc.Siteid = 'TRANS'
       AND v_ServiceAddress_bc.SiteId = 'TRANS'
       AND v_Locations_bc."Status" = 'IN SERVICE'
       AND Etrmrid IS NOT NULL
"""

def getGMSData():
   stationDF = pd.read_csv(GMS_STATIONS_DATA)
   lineDF = pd.read_csv(GMS_LINES_DATA)

   # Replace spaces with underscores for all column names (prevents pandas setting them to weird names)
   stationDF.columns = [c.replace(' ', '_') for c in stationDF.columns]
   lineDF.columns = [c.replace(' ', '_') for c in lineDF.columns]

   return (stationDF, lineDF)

def getTOMSData():
   # Returns Substation Data from TOMS
   with pyodbc.connect(TOMS_CONN_STR) as conn:
       return pd.read_sql(TOMS_SUB_QUERY, conn)

def getMaximoData():
   # Returns Substation Data from Maximo
   with pyodbc.connect(MAXIMO_CONN_STR) as conn2:
       return pd.read_sql(MAXIMO_SUB_QUERY, conn2)

def getTieStationData():
   tiestationDF = pd.read_excel(TIESTATIONS_DATA)
   tiestationDF.columns = [c.replace(' ', '_') for c in tiestationDF.columns]
   return tiestationDF

# Takes the third word of CimPathName, matches the GMS ID. Removes outages not found in graph.
#  THIS FUNCTION WILL NOT BE NECESSARY ONCE CimPathName MATCHES BETWEEN GMS AND TOMS
def cleanOutages(outageDF, grid):
   outageDF['CimID'] = outageDF['CimID'].str.split(' ').str[2]
   isValid = outageDF['CimID'].notna() & outageDF['CimID'].isin(grid.lineSegments)
   missingCim = len(outageDF) - isValid.sum()

   if missingCim > 0:
       print(f'WARNING: {missingCim} outaged line segments CimPathName missing or null out of all outages.')
   return outageDF[isValid].copy()

def getOutages():
   # Returns DF of all planned and unplanned outages
   outageDF = None
   with pyodbc.connect(TOMS_CONN_STR) as conn:
       outageDF =  pd.read_sql(outagesQuery, conn)
   if ONLY_LINES_OUTAGES:
       outageDF = outageDF[outageDF['Device Type'] == 'Line']
   return outageDF

def getStormWindow(outageDF):
   startTime = None
   endTime = None
   for outage in outageDF.itertuples(index=False):
       if pd.isna(outage.StormName) or outage.StormName != STORM_NAME: continue

       # Set startTime and endTime if this is the first found outage associated with storm
       if startTime == None:
           startTime = outage.Implemented
           endTime = outage.Complete
           continue

       # Get the earliest start time
       if startTime > outage.Implemented:
           startTime = outage.Implemented

       # Get the latest start time for completed storms, set up to return None if any outage does not have a complete time for ongoing storms
       if endTime == None: continue
       if pd.isna(outage.Complete):
           endTime = None
       elif endTime < outage.Complete:
           endTime = outage.Complete

   if endTime == None:
       endTime = pd.Timestamp.now()
   # Raise error if storm start time is not detected
   if startTime == None:
       raise ValueError('STORM START TIME IS NOT DETECTED, CHECK STORM_NAME SPELLING')
   
   return startTime, endTime

def getOutageEvents(outageDF, startTime, endTime):
   # Filter outages to only outages occuring during storm
   toKeep = []
   for outage in outageDF.itertuples():
       # Gets ongoing outages if they started during the storm window
       if pd.isna(outage.Complete) and outage.Implemented <= endTime:
           toKeep.append(outage)
           continue
       elif pd.isna(outage.Complete) and outage.Implemented > endTime:
           continue

       # Gets all outages that ended or started during the storm window
       if outage.Implemented <= endTime and outage.Complete >= startTime:
           toKeep.append(outage)
   if len(toKeep) < 1:
       raise ValueError('NO OUTAGES WITHIN STORM WINDOW')

   stormOutages = pd.DataFrame(toKeep)
   stormOutages = stormOutages.rename(columns={'PlanNum':'OutageID', 'ItemNum':'ItemID'})

   # Create a dataframe for oosEvents and rtsEvents, remove rtsEvents with a NULL Completition time (these outages have not been returned to service)
   oosEvents = stormOutages[['OutageID', 'ItemID', 'CimID', 'Implemented']].rename(columns={'Implemented': 'Time'})
   oosEvents['EventType'] = 'OOS'
   rtsEvents = stormOutages[['OutageID', 'ItemID', 'CimID', 'Complete']].rename(columns={'Complete': 'Time'})
   rtsEvents = rtsEvents[rtsEvents['Time'].notna()]
   rtsEvents['EventType'] = 'RTS' 

   # Combine the events into one dataframe and sort them based on implemented/complete time
   outageEvents = pd.concat([oosEvents, rtsEvents]).sort_values(by='Time')
   # Return as list of named tuples instead of dictionaries (so I can use dot notation)
   return list(outageEvents.itertuples(index=False))

class Island():

   def __init__(self, energized):
       self.stations = []
       self.lines = []
       self.energized = energized

class Station():

   def __init__(self, cimID):
       # Main station data
       self.CIMID = cimID
       self.MRID = None
       self.description = None
       self.county = None
       self.jurisdiction = None
       self.company = None
       self.voltage = None

       #Grid data
       self.lineSegments = []
       self.energized = True
       self.lastChanged = None
       self.isMaximo = False
       self.isTieStation = False

class LineSegment():

   def __init__(self, cimID, fromm, to, kv):
       # Main line data
       self.CIMID = cimID
       self.fromStation = fromm
       self.toStation = to
       self.voltage = kv

       # Grid data
       self.inService = True
       self.energized = True
       self.lastChanged = None

class Grid():

   def __init__(self):
       self.stations = {}
       self.lineSegments = {}
       self.tieStations = []

   def createSubs(self, stationDF):
       for stationData in stationDF.itertuples(index=False):
           # Create a Station object for each unique GMS station
           if stationData.Station not in self.stations:
               newStation = Station(stationData.Station)
               self.stations[newStation.CIMID] = newStation
       
       # Get MRIDs from TOMS
       tomsDF = getTOMSData()
       for stationData in tomsDF.itertuples(index=False):
           if not pd.isna(stationData.CimPathName) and stationData.CimPathName in self.stations:
               self.stations[stationData.CimPathName].MRID = stationData.MRID.upper()

       # Gets Reporting Data from Maximo
       maximoDF = getMaximoData()  
       for stationData in maximoDF.itertuples(index=False):
           # Attempt to match station using both ETRMRIDNAME and MRID
           matchedStation = None
           if not pd.isna(stationData.ETRMRIDNAME) and stationData.ETRMRIDNAME in self.stations:
               matchedStation = self.stations[stationData.ETRMRIDNAME]
           elif not pd.isna(stationData.MRID):
               for station in self.stations.values():
                   if station.MRID == None: continue
                   if stationData.MRID.upper() == station.MRID.upper():
                       matchedStation = station
                       break
                   
           # Store Maximo data in the matching station
           if matchedStation != None:
               matchedStation.isMaximo = True
               matchedStation.description = stationData.Description
               matchedStation.county = stationData.County
               matchedStation.company = stationData.EtrSubstnCompany
               matchedStation.voltage = stationData.EtrVoltage1

               # Get Jurisdiction using dict
               if stationData.EtrGrid in JURISDICTION_BY_GRID:
                   matchedStation.jurisdiction = JURISDICTION_BY_GRID[stationData.EtrGrid]

   def createLines(self, lineDF):
       for lineData in lineDF.itertuples(index=False):
           # Only create line segments if they have not been created and its from and to station are in the graph (also must be a line or ZBR)
           if lineData.ID not in self.lineSegments:
               if lineData.Equipment.upper() not in ('LN', 'ZBR'): continue
               if lineData.Station not in self.stations or lineData.To_Station not in self.stations: continue

               newLine = LineSegment(lineData.ID, self.stations[lineData.Station], self.stations[lineData.To_Station], lineData.KV_Nominal)
               self.lineSegments[newLine.CIMID] = newLine

               # Store line in associated stations
               newLine.fromStation.lineSegments.append(newLine)
               newLine.toStation.lineSegments.append(newLine)

   def assignTieStations(self):
       tiestationDF = getTieStationData()
       tiestationMRIDs = set()
       for tiestation in tiestationDF.itertuples(index=False):
           tiestationMRIDs.add(tiestation.MRID.upper())
   
       foundStations = []
       for station in self.stations.values():
           if station.MRID == None: continue
           if station.MRID in tiestationMRIDs:
               station.isTieStation = True
               foundStations.append(station)
       for station in foundStations:
           self.tieStations.append(station)

   def pruneIslands(self):
       islands = self.getIslands()
       for island in islands:
           if island.energized: continue
           # Remove stations and lines that are not energized 
           for station in island.stations:
               if station.CIMID in self.stations:
                   del self.stations[station.CIMID]
           for line in island.lines:
               if line.CIMID in self.lineSegments:
                   del self.lineSegments[line.CIMID]

   def generateTopology(self):
       # Get the GMS Station data and build objects representing them
       stationDF, lineDF = getGMSData()
       self.createSubs(stationDF)
       self.createLines(lineDF)

       # Get and assign tie stations
       self.assignTieStations()

       # If setting set to True, prune non-energized lines and stations (elements outside of the main grid)
       if PRUNE_ISLANDS:
           self.pruneIslands()

   def getNeighborStations(self, station, visited, excludeOOS=True):
       stations = []
       lines = []
       # Get lines we have not visited yet, along with their substations (if line is in service)
       for line in station.lineSegments:
           if excludeOOS and not line.inService: continue
           if line.CIMID in visited: continue
           visited.add(line.CIMID)
           lines.append(line)

           #check to ensure station has not been visited
           neighbor = line.fromStation if line.fromStation.CIMID != station.CIMID else line.toStation
           if neighbor.CIMID not in visited:
               visited.add(neighbor.CIMID)
               stations.append(neighbor)

       return stations, lines
   
   def outageBFS(self, station):
       # Traverse the grid from the given station to determine if the given station along with reachable stations are energized or not
       energized = False
       traversedStations = [station]

       visited = {station.CIMID}
       queue = deque([station])
       while queue:
           currentStation = queue.popleft()
           if currentStation.isTieStation:
               energized = True

           stations, lines = self.getNeighborStations(currentStation, visited)
           queue.extend(stations)
           traversedStations.extend(stations)

       return traversedStations, energized

   def oosEvent(self, line, event):
       logs = []
       # New outages only occur when both stations attached to line were inService before line was taken OOS
       # If outage station occurs it will only happen for one side of the segment
       stationsOOS = []
       if line.fromStation.energized and line.toStation.energized:
           stations, energized = self.outageBFS(line.fromStation)
           if not energized:
               stationsOOS.extend(stations)
           else:
               stations,energized = self.outageBFS(line.toStation)
               if not energized:
                   stationsOOS.extend(stations)
       # Mark stations as de-energized and Create OOS Logs
       for station in stationsOOS:
           station.energized = False
           if not REPORT_TOMS_STATIONS and not station.isMaximo: continue
           log = {'outageID': event.OutageID, 'itemID': event.ItemID, 'station': station, 'time': event.Time, 'logType': 'OOS'}
           logs.append(log)
       return logs

   def rtsEvent(self, line, event):
       logs = []
       stationsRTS = []

       # If one station is in service and the other is not, the OOS station will be returned to service by the line being RTS
       rootRTS = None
       if line.fromStation.energized and not line.toStation.energized:
           rootRTS = line.toStation
       elif not line.fromStation.energized and line.toStation.energized:
           rootRTS = line.fromStation

       # Get stations attached to newly rts station since they will also be rts
       if rootRTS != None:
           stations, e = self.outageBFS(rootRTS)
           stationsRTS.extend(stations)

       # Mark stations as energized and Create RTS Logs
       line.inService = True
       for station in stationsRTS:
           station.energized = True
           if not REPORT_TOMS_STATIONS and not station.isMaximo: continue
           log = {'outageID': event.OutageID, 'itemID': event.ItemID, 'station': station, 'time': event.Time, 'logType': 'RTS'}
           logs.append(log)
       return logs
   
   def simulateOutages(self, outageDF):
       # Clean outages and get outage events associated with the storm
       outageDF = cleanOutages(outageDF, self)
       startTime, endTime = getStormWindow(outageDF)
       outageEvents = getOutageEvents(outageDF, startTime, endTime)

       #Simulate the events happening over time, create logs
       logs = []
       for event in outageEvents:
           line = self.lineSegments[event.CimID]
           if event.EventType == 'OOS':
               line.inService = False
               logs.extend(self.oosEvent(line, event))
           elif event.EventType == 'RTS':
               logs.extend(self.rtsEvent(line, event))
       return logs
   
   def islandDFS(self, islands, visited, outer, energized):
       outerQueue = deque(outer)
       while outerQueue:
           # Outer loop used to create new islands
           rootStation = outerQueue.popleft()
           if rootStation.CIMID in visited: continue

           newIsland = Island(energized)
           newIsland.stations.append(rootStation)
           visited.add(rootStation.CIMID)
           # Inner loop used to populate island
           innerQueue = deque([rootStation])
           while innerQueue:
               currentStation = innerQueue.popleft()

               # Get lines and neighbor stations, add them to island
               stations, lines = self.getNeighborStations(currentStation, visited, energized)
               newIsland.stations.extend(stations)
               newIsland.lines.extend(lines)
               innerQueue.extendleft(stations)

               if len(innerQueue) == 0:
                   islands.append(newIsland)

   def getIslands(self):
       islands = []
       visited = set()

       # Build energized Islands first
       self.islandDFS(islands, visited, self.tieStations, True)

       # Get list of stations that have not been put in an island (these are non-energized)
       unvisitedStations = []
       for station in self.stations.values():
           if station.CIMID not in visited:
               unvisitedStations.append(station)
       # Build non-energized Islands
       self.islandDFS(islands, visited, unvisitedStations, False)

       return islands
   
   def getSubstationsOOS(self):
       count = 0
       for station in self.stations.values():
           if station.energized: continue
           if not REPORT_TOMS_STATIONS and not station.isMaximo: continue
           count += 1
       return count
       
def main():
   grid = Grid()
   grid.generateTopology()
   outageDF = getOutages()
   logs = grid.simulateOutages(outageDF)

   islands = grid.getIslands()
   print(grid.getSubstationsOOS())

main()