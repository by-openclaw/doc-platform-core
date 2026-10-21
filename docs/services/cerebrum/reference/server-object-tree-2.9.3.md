# Cerebrum server — object tree of device `0.0.0.0` (release 2.9.3)

Generated 2026-10-22 from the vendor's device-definition files (`GenericDevices\*.cgd`) exported from the lab server. One table per sub-device of the server device; the path is the `OBJECT` of a Northbound `DEVICE_CHANGE TYPE="VALUE"` obtain or subscribe with `IP_ADDRESS="0.0.0.0"` and `SUB_DEVICE` = the index below. Tables (`TABLE`, `DYNAMIC_TABLE`) are indexed `Name.[n]` or wildcarded `Name.[*]` on the wire. Access: R read-only, W writable (a control), blank = not declared. Severities are the vendor's own state → severity mapping (`Good`, `Warning`, `Error`, `Info`, `Ignore`), the basis of the "running fine" rules in the README §14.

Do not edit by hand: regenerate from the definition files when the release changes.

## `00: System` — definition `Application Overview` (298 nodes)

| Path | Kind | Access | Units / range | Enum | Severities | Description |
|---|---|---|---|---|---|---|
| `ComputerOverview` | GROUP |  |  |  |  |  |
| &nbsp;&nbsp;`ProcessorTime` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;`ProcessCount` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;`ThreadCount` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;`HandleCount` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;`Memory` | GROUP |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`PhysicalInUse` | VALUE/UINT32 | R | MB |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`PhysicalAvailable` | VALUE/UINT32 | R | MB |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`CommittedTotal` | VALUE/UINT32 | R | MB |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`CommittedPeak` | VALUE/UINT32 | R | MB |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`CommittedLimit` | VALUE/UINT32 | R | MB |  |  |  |
| &nbsp;&nbsp;`Network` | GROUP |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Transmit` | GROUP |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`BytesPerSec` | VALUE/UINT32 | R | Bytes/sec |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`BytesPerSecMax` | VALUE/UINT32 | R | Bytes/sec |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`PacketsPerSec` | VALUE/UINT32 | R | Packets/sec |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`PacketsPerSecMax` | VALUE/UINT32 | R | Packets/sec |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Receive` | GROUP |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`BytesPerSec` | VALUE/UINT32 | R | Bytes/sec |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`BytesPerSecMax` | VALUE/UINT32 | R | Bytes/sec |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`PacketsPerSec` | VALUE/UINT32 | R | Packets/sec |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`PacketsPerSecMax` | VALUE/UINT32 | R | Packets/sec |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`ResetPeakValues` | VALUE/BOOL |  |  |  |  |  |
| `ApplicationOverview` | GROUP |  |  |  |  |  |
| &nbsp;&nbsp;`ProcessorTime` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;`HandleCount` | VALUE/UINT32 | R |  |  | 0..50000=Good; 50001..60000=Warning; 60001..200000=Error |  |
| &nbsp;&nbsp;`PeakHandleCount` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;`GDIObjects` | VALUE/UINT32 | R |  |  | 0..8000=Good; 8001..9000=Warning; 9001..10000=Error |  |
| &nbsp;&nbsp;`PeakGDIObjects` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;`UserObjects` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;`PeakUserObjects` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;`DeviceDeactivations` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;`Memory` | GROUP |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`WorkingSet` | VALUE/UINT32 | R | MB |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`PeakWorkingSet` | VALUE/UINT32 | R | MB |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`CommitSize` | VALUE/UINT32 | R | MB |  | 0..5000=Good; 5001..7500=Warning; 7501..16384=Error |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`PeakCommitSize` | VALUE/UINT32 | R | MB |  |  |  |
| `InternalMemoryManagement` | GROUP |  |  |  |  |  |
| &nbsp;&nbsp;`TotalReservedSize` | VALUE/UINT32 | R | MB |  |  |  |
| &nbsp;&nbsp;`AllocatedBlocks` | TABLE |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`BlockSize` | VALUE/UINT32 | R | MB |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`CountInUse` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`CountFree` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`CountFree` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`AccessCount` | VALUE/UINT32 | R |  |  |  |  |
| `InternalQueues` | GROUP |  |  |  |  |  |
| &nbsp;&nbsp;`MessageQueueLength` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;`PeakMessageQueueLength` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;`MessageQueue` | GROUP |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`PeakBacklogAge` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`PeakBacklogUser` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`CurrentBacklogAge` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`CurrentBacklogUser` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;`TotalMessageQueueLength` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;`EventLogQueueLength` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;`PeakEventLogQueueLength` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;`TotalEventLogQueueLength` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;`LastEventLogCommitDuration` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;`AuditTrailQueueLength` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;`PeakAuditTrailQueueLength` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;`TotalAuditTrailQueueLength` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;`CorLinkSendQueue` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;`PeakCorLinkSendQueue` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;`TotalCorLinkSendQueue` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;`CorLinkReceiveQueue` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;`PeakCorLinkReceiveQueue` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;`TotalCorLinkReceiveQueue` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;`CorLinkRate` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;`ResetPeakQueueLengths` | VALUE/BOOL |  |  |  |  |  |
| &nbsp;&nbsp;`WriteCount` | VALUE/UINT32 | R |  |  |  |  |
| `ThreadResponsiveness` | GROUP |  |  |  |  |  |
| &nbsp;&nbsp;`LastGUIDelay` | VALUE/UINT32 | R | ms |  |  |  |
| &nbsp;&nbsp;`PeakGUIDelay` | VALUE/UINT32 | R | ms |  |  |  |
| &nbsp;&nbsp;`MacroThreads` | TABLE |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`LastThreadDelay` | VALUE/UINT32 | R | ms |  | 0..2000=Good; 2001..5000=Warning; 5001..16384000=Error |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`PeakThreadDelay` | VALUE/UINT32 | R | ms |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`ThreadType` | VALUE/UINT8 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`AssociatedMacros` | VALUE/STRING | R |  |  |  |  |
| `Redundancy_Status` | GROUP |  |  |  |  |  |
| &nbsp;&nbsp;`Is_Redundant_System` | VALUE/BOOL | R |  |  |  |  |
| &nbsp;&nbsp;`Overall` | VALUE/UINT8 | R |  |  | OK=Good; Error=Error |  |
| &nbsp;&nbsp;`Active_Server` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;`Cerebrum_Partner` | VALUE/UINT8 | R |  |  | OK=Good; Error=Error |  |
| &nbsp;&nbsp;`Cerebrum_Witness` | VALUE/UINT8 | R |  |  | OK=Good; Error=Error |  |
| &nbsp;&nbsp;`SQL_Partner` | VALUE/UINT8 | R |  |  | OK=Good; Synchronising=Warning; Error=Error |  |
| &nbsp;&nbsp;`SQL_Witness` | VALUE/UINT8 | R |  |  | OK=Good; Error=Error |  |
| &nbsp;&nbsp;`SQL_State` | VALUE/UINT8 | R |  |  | Online=Good; Restoring=Good; Recovering=Good; RecoveryPending=Warning; Emergency=Error; Offline=Error |  |
| &nbsp;&nbsp;`SQL_Log_Maintenance` | VALUE/UINT8 | R |  |  | OK=Good; Error=Error |  |
| &nbsp;&nbsp;`Watchdog_Health` | VALUE/UINT8 | W | 0..100 |  |  |  |
| `Federation` | GROUP |  |  |  |  |  |
| &nbsp;&nbsp;`Is_Using_Federation_Service` | VALUE/BOOL | R |  |  |  |  |
| &nbsp;&nbsp;`Overall` | VALUE/UINT8 | R |  |  | OK=Good; Error=Error |  |
| &nbsp;&nbsp;`Failover` | VALUE/BOOL |  |  |  |  |  |
| &nbsp;&nbsp;`Primary` | GROUP |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Server_FQDN` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Server_State` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Server_Version` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;`Secondary` | GROUP |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Server_FQDN` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Server_State` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Server_Version` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;`Connected_Servers` | DYNAMIC_TABLE |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Server_Name` | VALUE/STRING | R |  |  |  |  |
| `RouteMaster` | GROUP |  |  |  |  |  |
| &nbsp;&nbsp;`TieLines` | GROUP |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`ConfigCount` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`ActiveCount` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Shufflers` | GROUP |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`Rx_Defined` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`Tx_Defined` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`Tx_In_Use` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Usage` | DYNAMIC_TABLE |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`StartDevice` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`StartTypeID` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`StartLevel` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`StartOutput` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`StartReceiver` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`StartSubChan` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`EndDevice` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`EndTypeID` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`EndLevel` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`EndInput` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`EndSender` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`EndSubChan` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`Reserved_Dest` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`InUse_Dest` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Errors` | GROUP |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`Count` | VALUE/UINT32 | R |  |  | *=Info |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`CountReset` | VALUE/BOOL |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`LastErrorMessage` | VALUE/STRING | R |  |  | *=Info |  |
| `ControlInterfaces` | GROUP |  |  |  |  |  |
| &nbsp;&nbsp;`Name` | DYNAMIC_TABLE |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Connection` | DYNAMIC_TABLE |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`Present` | VALUE/BOOL | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`RxActive` | VALUE/BOOL | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`TxActive` | VALUE/BOOL | R |  |  |  |  |
| `License` | GROUP |  |  |  |  |  |
| &nbsp;&nbsp;`Sentinel_Key_Serial_Number` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;`License_Type` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;`Maintenance_Support_Expiry_Date` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;`Demo_License_Enabled` | VALUE/UINT8 | R |  |  |  |  |
| &nbsp;&nbsp;`Demo_License_Duration` | VALUE/STRING | R | Minutes |  |  |  |
| &nbsp;&nbsp;`Demo_License_Remaining` | VALUE/STRING | R | Minutes |  | 0..240=Error |  |
| &nbsp;&nbsp;`Supported_Options` | GROUP |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Client_Licenses` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Lite_Client_Licenses` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Web_Client_Licenses` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Panel_View` | VALUE/UINT8 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Redundant_Server` | VALUE/UINT8 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Router_Control` | VALUE/UINT8 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Route_Master` | VALUE/UINT8 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Route_Master_Destination_Limit` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Route_Master_Destinations_In_Use` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`IP_Destination_Limit` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`IP_Destinations_In_Use` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`IP-Core` | VALUE/UINT8 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`IP_Shuffle_Limit` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`IP_Shuffle_In_Use` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Max_IP_Destination_ID` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Tielines` | VALUE/UINT8 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Salvos` | VALUE/UINT8 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`SNMP` | VALUE/UINT8 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`SNMP_Device_Limit` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Tally_Manager` | VALUE/UINT8 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Generic_Device_Points` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Generic_Device_Points_In_Use` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Northbound_API_Licenses` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Northbound_API_Licenses_In_Use` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Resource_Manager_Licenses` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Resource_Manager_Licenses_In_Use` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Assignment_Engine_Licenses` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Assignment_Engine_Licenses_In_Use` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Max_Package_Router` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Package_Router_In_Use` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`External_Logging` | VALUE/UINT8 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Federation` | VALUE/UINT32 | R |  |  |  |  |
| `ConfigDatabase` | GROUP |  |  |  |  |  |
| &nbsp;&nbsp;`IndependentConfigState` | VALUE/UINT8 | R |  |  | *=Info; Unknown=Error; Differ=Warning; Same=Good; Not Connected=Error |  |
| &nbsp;&nbsp;`PushToPartner` | VALUE/BOOL |  |  |  | True=Info; *=Ignore |  |
| &nbsp;&nbsp;`PullFromPartner` | VALUE/BOOL |  |  |  | True=Info; *=Ignore |  |
| `ServerFileTransfer` | GROUP |  |  |  |  |  |
| &nbsp;&nbsp;`PushToPartner` | VALUE/BOOL |  |  |  | True=Info; *=Ignore |  |
| &nbsp;&nbsp;`Progress` | VALUE/UINT8 | R |  |  | True=Info; *=Ignore |  |
| `SQLAlwaysOnStatus` | GROUP |  |  |  |  |  |
| &nbsp;&nbsp;`GroupName` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;`GroupId` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;`Server` | DYNAMIC_TABLE |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`ReplicaServerName` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`ReplicaId` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Role` | VALUE/UINT8 | R | 0..2 |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`UpToDate` | VALUE/BOOL | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`AlterAvailabilityGroup` | VALUE/UINT8 |  | 0..1 |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`OperationalState` | VALUE/UINT8 | R | 0..5 |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`RecoveryHealth` | VALUE/UINT8 | R | 0..1 |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`SynchronizationState` | VALUE/UINT8 | R | 0..4 |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`SynchronizationHealth` | VALUE/UINT8 | R | 0..2 |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`ConnectedState` | VALUE/UINT8 | R | 0..1 |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`DatabaseState` | VALUE/UINT8 | R | 0..6 |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`IsSuspended` | VALUE/UINT8 | R | 0..1 |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`SuspendReason` | VALUE/UINT8 | R | 0..8 |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`LogSendRate` | VALUE/UINT64 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`FilestreamSendRate` | VALUE/UINT64 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`SecondaryLagSeconds` | VALUE/UINT64 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`LastConnectErrorNumber` | VALUE/UINT8 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`LastConnectErrorTimestamp` | VALUE/STRING | R |  |  |  |  |
| `LoggingServers` | GROUP |  |  |  |  |  |
| &nbsp;&nbsp;`LoggingServer` | DYNAMIC_TABLE |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`ServerType` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`IPAddressOrHost` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Status` | VALUE/UINT8 | R |  |  |  |  |
| `Clients` | GROUP |  |  |  |  |  |
| &nbsp;&nbsp;`Client` | DYNAMIC_TABLE |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`ClientType` | VALUE/UINT8 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`IPAddress` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Name` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`MacAddress` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Port` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Licence` | VALUE/UINT8 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`User` | VALUE/STRING | W |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Browser` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`JoinTime` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`State` | VALUE/UINT8 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`LogOff` | VALUE/BOOL | W |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Access` | VALUE/UINT8 | W |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`MessageQueueLength` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`PeakMessageQueueLength` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`MessageQueueAge` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`PeakMessageQueueAge` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`MessageCount` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`CombinedMessageCount` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`ResetStatistics` | VALUE/BOOL | W |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Host` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`FileSync` | VALUE/UINT8 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`SoftwareVersion` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Drop` | VALUE/BOOL | W |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Upgrade` | VALUE/BOOL | W |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`WebClientEnabled` | VALUE/BOOL | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`HardwarePanelsEnabled` | VALUE/BOOL | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`HardwarePanelDetails` | DYNAMIC_TABLE |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`IP` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`TypeID` | VALUE/UINT64 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`ConnectionStatus` | VALUE/UINT8 | W |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`UID` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`FriendlyName` | VALUE/STRING | W |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`AutoLogin` | VALUE/BOOL | W |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`RoutedViaProxy` | VALUE/BOOL | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`ChangeRequestsReceived` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`DataRequestsReceived` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Latency` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Resends` | VALUE/UINT64 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Status` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;`LicencedCount` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;`LogOff` | VALUE/BOOL | W |  |  |  |  |
| &nbsp;&nbsp;`PeakMessageQueueLength` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;`PeakMessageQueueAge` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;`ReceivedMessageCount` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;`ResetStatistics` | VALUE/BOOL | W |  |  |  |  |
| &nbsp;&nbsp;`RichClientCount` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;`UpgradeClientToVersion` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;`RichClientStatus` | VALUE/BOOL | W |  |  |  |  |
| &nbsp;&nbsp;`ChangeAccessControlList` | FUNCTION |  |  |  |  |  |
| &nbsp;&nbsp;`AccessControlList` | DYNAMIC_TABLE |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`IPAddress` | VALUE/STRING | W |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Access` | VALUE/UINT8 | W |  |  |  |  |
| `CorLinkStats` | GROUP |  |  |  |  |  |
| &nbsp;&nbsp;`TX` | GROUP |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`DataPackets` | VALUE/UINT64 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Acks` | VALUE/UINT64 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Nacks` | VALUE/UINT64 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`TimeOuts` | VALUE/UINT64 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Resends` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Size` | VALUE/UINT64 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Compression` | VALUE/FLOAT | R |  |  |  |  |
| &nbsp;&nbsp;`RX` | GROUP |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`DataPackets` | VALUE/UINT64 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Acks` | VALUE/UINT64 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Nacks` | VALUE/UINT64 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Resends` | VALUE/UINT64 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Size` | VALUE/UINT64 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Compression` | VALUE/FLOAT | R |  |  |  |  |
| &nbsp;&nbsp;`ChangeRequestsReceived` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;`DataRequestsReceived` | VALUE/UINT32 | R |  |  |  |  |
| &nbsp;&nbsp;`ResetStats` | VALUE/BOOL | W |  |  |  |  |
| `Users` | GROUP |  |  |  |  |  |
| &nbsp;&nbsp;`User` | DYNAMIC_TABLE |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Group` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`ControlCLF` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`NavigationCLF` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`UserType` | VALUE/UINT8 | R |  |  |  |  |
| `Foundation` | GROUP |  |  |  |  |  |
| &nbsp;&nbsp;`Enabled` | VALUE/BOOL | R |  |  |  |  |
| &nbsp;&nbsp;`ExternalProxyFQDN` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;`KeycloakAvailable` | VALUE/BOOL | R |  |  |  |  |
| &nbsp;&nbsp;`KeycloakCertCurrent` | VALUE/BOOL | R |  |  |  |  |
| &nbsp;&nbsp;`WebClientServiceIPAddresses` | TABLE |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`IPAddress` | VALUE/STRING |  |  |  |  |  |
| &nbsp;&nbsp;`ConfigModified` | VALUE/BOOL | R |  |  |  |  |
| &nbsp;&nbsp;`RestartWebServer` | VALUE/BOOL |  |  |  |  |  |

## `01: IP Router` — definition `IPRouting` (52 nodes)

| Path | Kind | Access | Units / range | Enum | Severities | Description |
|---|---|---|---|---|---|---|
| `Senders` | DYNAMIC_TABLE |  |  |  |  |  |
| &nbsp;&nbsp;`Device` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;`UID` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;`DeviceLabel` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;`Label` | VALUE/STRING |  |  |  |  |  |
| &nbsp;&nbsp;`ControlType` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;`FromDevice` | VALUE/BOOL | R |  |  |  |  |
| &nbsp;&nbsp;`SignalType` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;`Available` | VALUE/BOOL | R |  |  |  |  |
| &nbsp;&nbsp;`SenderStatus` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;`Fed_By_Output` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;`Stream_Status` | TABLE |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Origin` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Multicast` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Port` | VALUE/UINT16 | R |  |  |  |  |
| &nbsp;&nbsp;`NMS_ID_1` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;`NMS_ID_2` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;`FlowClashes` | VALUE/UINT8 | R |  |  |  |  |
| &nbsp;&nbsp;`SubChannels` | VALUE/STRING | R |  |  |  |  |
| `Receivers` | DYNAMIC_TABLE |  |  |  |  |  |
| &nbsp;&nbsp;`Device` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;`UID` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;`DeviceLabel` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;`Label` | VALUE/STRING |  |  |  |  |  |
| &nbsp;&nbsp;`ControlType` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;`FromDevice` | VALUE/BOOL | R |  |  |  |  |
| &nbsp;&nbsp;`SignalType` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;`Available` | VALUE/BOOL | R |  |  |  |  |
| &nbsp;&nbsp;`ReceiverStatus` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;`Current_Sender` | VALUE/STRING |  |  |  |  |  |
| &nbsp;&nbsp;`Requested_Sender` | VALUE/STRING |  |  |  |  |  |
| &nbsp;&nbsp;`Stream_Status` | TABLE |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Origin` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Multicast` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Port` | VALUE/UINT16 | R |  |  |  |  |
| &nbsp;&nbsp;`Stream_Invalid` | VALUE/BIT_PAT | R |  |  |  |  |
| &nbsp;&nbsp;`Feeding_Input` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;`NMS_ID_1` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;`NMS_ID_2` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;`SubChannels` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;`NAT_SDP` | VALUE/STRING | R |  |  |  |  |
| `SenderProfiles` | DYNAMIC_TABLE |  |  |  |  |  |
| &nbsp;&nbsp;`SDP` | VALUE/STRING |  |  |  |  |  |
| `NMS` | GROUP |  |  |  |  |  |
| &nbsp;&nbsp;`Senders` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;`Receivers` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;`IP_Address` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;`TypeID` | VALUE/INT32 | R |  |  |  |  |
| &nbsp;&nbsp;`Type` | VALUE/UINT8 | R |  |  |  |  |
| `Engineering` | GROUP |  |  |  |  |  |
| &nbsp;&nbsp;`AbortRouteRequests` | VALUE/BOOL |  |  |  |  |  |
| &nbsp;&nbsp;`CleanReceiverRoutes` | VALUE/BOOL |  |  |  |  |  |

## `02: Services` — definition `Services` (98 nodes)

| Path | Kind | Access | Units / range | Enum | Severities | Description |
|---|---|---|---|---|---|---|
| `ALL_RESPONSIVE` | VALUE/BOOL | R |  |  |  |  |
| `SERVERS` | DYNAMIC_TABLE |  |  |  |  |  |
| &nbsp;&nbsp;`LABEL` | VALUE/STRING |  |  |  |  |  |
| &nbsp;&nbsp;`IS_REMOVABLE` | VALUE/BOOL | R |  |  |  |  |
| &nbsp;&nbsp;`REMOVE` | VALUE/BOOL | W |  |  |  |  |
| &nbsp;&nbsp;`SEND_EXTERNAL_LOG_CONFIG` | VALUE/BOOL | W |  |  |  |  |
| &nbsp;&nbsp;`CLEAR_SHUTDOWN_STATUS` | VALUE/BOOL | W |  |  |  |  |
| &nbsp;&nbsp;`SERVICES` | DYNAMIC_TABLE |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`DISPLAY_NAME` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`STATE` | VALUE/STRING | R |  |  | Running=Good; Start Pending=Warning; Stop Pending=Warning; Stopped=Error; Unknown=Error |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`TYPE` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`START` | VALUE/BOOL | W |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`STOP` | VALUE/BOOL | W |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`DELETE` | VALUE/BOOL | W |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`RESPONSIVE` | VALUE/BOOL | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`DATA_PORT` | VALUE/UINT | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`CAN_BE_DELETED` | VALUE/BOOL | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`ENTER_DEBUG_MODE` | VALUE/BOOL | W |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`SEND_EXTERNAL_LOG_CONFIG` | VALUE/BOOL | W |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`FAILED_HTTP_REQUESTS` | VALUE/UINT | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`LAST_HTTP_ERROR` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`EXTERNAL_LOGGING_SERVERS` | DYNAMIC_TABLE |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`TYPE` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`ADDRESS` | VALUE/STRING |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`PORT` | VALUE/UINT |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`MESSAGES_TO_LOG` | VALUE/UINT8 |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`TRANSPORT_TYPE` | VALUE/UINT8 |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`STATUS` | VALUE/UINT8 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`SERVICE_INFORMATION` | DYNAMIC_TABLE |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`VALUE` | VALUE/STRING |  |  |  |  |  |
| &nbsp;&nbsp;`DEVICE_SERVICES` | DYNAMIC_TABLE |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`DEVICES` | DYNAMIC_TABLE |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`IP_ADDRESS` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`SECONDARY_IP` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`PORT` | VALUE/UINT | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`STREAM_TYPE` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`DEVICE_NAME` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`PROTOCOL` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`PRIMARY_CONNECTION_STATE` | VALUE/INT8 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`SECONDARY_CONNECTION_STATE` | VALUE/INT8 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`MASTER_SERVER_STATE` | VALUE/INT8 | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`ALL_SUSCRIPTIONS` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`ACTIVE_SUBSCRIPTIONS` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`ACTIVE_SUBSCRIBER_NAMES` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`MASTER_SERVER` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`MASTER_SERVER_IP_ADDRESS` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`IS_IN_SYSTEM` | VALUE/BOOL | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`RECEIVING_UPDATES` | VALUE/BOOL | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`DEVICE_IS_ACTIVE` | VALUE/BOOL | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`SUBSCRIBE` | VALUE/BOOL | W |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`UNSUBSCRIBE` | VALUE/BOOL | W |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`FORCE_REMOVE` | VALUE/BOOL | W |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`FORCE_KEEP` | VALUE/BOOL | W |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`IS_FORCE_KEEP` | VALUE/BOOL | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`ACTIVATE_DEVICE` | VALUE/BOOL | W |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`DEACTIVATE_DEVICE` | VALUE/BOOL | W |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`FAILOVER` | VALUE/BOOL | W |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`FAILOVER_ENABLED` | VALUE/BOOL | W |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`SERVER_LAST_MESSAGE_SENT` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`SERVER_TOTAL_MESSAGES_SENT` | VALUE/UINT | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`SERVICE_LAST_MESSAGE_RECEIVED` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`SERVICE_TOTAL_MESSAGES_RECEIVED` | VALUE/UINT | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`SERVICE_LAST_MESSAGE_SENT` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`SERVICE_TOTAL_MESSAGES_SENT` | VALUE/UINT | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`SERVER_LAST_MESSAGE_RECEIVED` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`SERVER_TOTAL_MESSAGES_RECEIVED` | VALUE/UINT | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`DEVICE_LAST_THREAD_DELAY` | VALUE/UINT | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`DEVICE_PEAK_THREAD_DELAY` | VALUE/UINT | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`DATA_CONNECTION_HEALTH` | VALUE/INT8 | R |  |  | Good=Info; Delayed=Info; No Connection=Info |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`FORCE_REMOVE_ALL` | VALUE/BOOL | W |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`SERVER_LAST_MESSAGE_SENT` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`SERVER_TOTAL_MESSAGES_SENT` | VALUE/UINT | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`SERVICE_LAST_MESSAGE_RECEIVED` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`SERVICE_TOTAL_MESSAGES_RECEIVED` | VALUE/UINT | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`SERVICE_LAST_MESSAGE_SENT` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`SERVICE_TOTAL_MESSAGES_SENT` | VALUE/UINT | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`SERVER_LAST_MESSAGE_RECEIVED` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`SERVER_TOTAL_MESSAGES_RECEIVED` | VALUE/UINT | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`SERVER_LAST_THREAD_DELAY` | VALUE/UINT | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`SERVER_PEAK_THREAD_DELAY` | VALUE/UINT | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`CLEAR_AUDIT_MESSAGES` | VALUE/BOOL | W |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`FORWARD_AUDIT_MESSAGES` | VALUE/BOOL | W |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`RCP_CONFIGURATION` | GROUP |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`ENABLE_RCP` | VALUE/BOOL | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`ENABLE_SSL` | VALUE/BOOL | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`CERTIFICATE_PATH` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`CERTIFICATE_SERVER_KEY_PATH` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`CERTIFICATE_THUMBPRINT` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`HTTP_PORT` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`SET_CONFIGURATION` | FUNCTION |  |  |  |  |  |
| `SERVICE_CREATION` | GROUP |  |  |  |  |  |
| &nbsp;&nbsp;`SERVICE_NAME` | VALUE/STRING | W |  |  |  |  |
| &nbsp;&nbsp;`SERVICE_TYPE` | VALUE/INT8 | W |  |  |  |  |
| &nbsp;&nbsp;`SERVICE_LOCATION` | VALUE/STRING | W |  |  |  |  |
| &nbsp;&nbsp;`CREATE` | VALUE/BOOL | W |  |  |  |  |
| `ADD_SERVICE_MANAGER` | GROUP |  |  |  |  |  |
| &nbsp;&nbsp;`IP_ADDRESS` | VALUE/STRING | W |  |  |  |  |
| &nbsp;&nbsp;`ADD` | VALUE/BOOL | W |  |  |  |  |

## `03: Resource Manager` — definition `Resource Manager` (48 nodes)

| Path | Kind | Access | Units / range | Enum | Severities | Description |
|---|---|---|---|---|---|---|
| `Functions` | GROUP |  |  |  |  |  |
| &nbsp;&nbsp;`CreateAsset` | FUNCTION |  |  |  |  |  |
| &nbsp;&nbsp;`DeleteAsset` | FUNCTION |  |  |  |  |  |
| &nbsp;&nbsp;`CreateResource` | FUNCTION |  |  |  |  |  |
| &nbsp;&nbsp;`DeleteResource` | FUNCTION |  |  |  |  |  |
| &nbsp;&nbsp;`FindResources` | FUNCTION |  |  |  |  |  |
| &nbsp;&nbsp;`ReserveResource` | FUNCTION |  |  |  |  |  |
| &nbsp;&nbsp;`ModifyResourceReservation` | FUNCTION |  |  |  |  |  |
| &nbsp;&nbsp;`DeleteResourceReservation` | FUNCTION |  |  |  |  |  |
| `ResourceAvailableSource` | VALUE/STRING |  |  |  |  | The routemaster source which indicates a resource is not in use by route status |
| `AssetCount` | VALUE/UINT | R |  |  |  | The number of assets currently defined in the system |
| `ResourceCount` | VALUE/UINT | R |  |  |  | The number of resources currently defined in the system |
| `Assets` | DYNAMIC_TABLE |  |  |  |  |  |
| &nbsp;&nbsp;`Name` | VALUE/STRING |  |  |  |  |  |
| &nbsp;&nbsp;`CustomerUID` | GROUP |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Identifier` | VALUE/STRING |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`AssetID` | VALUE/STRING |  |  |  |  |  |
| &nbsp;&nbsp;`Device` | GROUP |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Manufacturer` | VALUE/STRING |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Model` | VALUE |  |  |  |  |  |
| &nbsp;&nbsp;`Location` | VALUE/STRING |  |  |  |  |  |
| &nbsp;&nbsp;`Description` | VALUE/STRING |  |  |  |  |  |
| &nbsp;&nbsp;`IPAddress` | VALUE/STRING |  |  |  |  |  |
| &nbsp;&nbsp;`DeviceName` | VALUE/STRING |  |  |  |  |  |
| &nbsp;&nbsp;`DeviceType` | VALUE/UINT16 |  |  |  |  |  |
| &nbsp;&nbsp;`SubID` | VALUE/STRING |  |  |  |  |  |
| &nbsp;&nbsp;`UserData` | GROUP |  |  |  |  |  |
| &nbsp;&nbsp;`Resources` | DYNAMIC_TABLE |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Name` | VALUE/STRING |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Type` | VALUE/STRING |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Functionality` | VALUE/UINT16 |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`UserInterfaces` | GROUP |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`MainUI` | VALUE/STRING |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`SmallUI` | VALUE/STRING |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`MonitoringUI` | VALUE/STRING |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`RouteMasterIO` | TABLE |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`Source` | VALUE/STRING |  |  |  |  | The numeric or mnemonic of the routemaster source associated with this device |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`Destination` | VALUE/STRING |  |  |  |  | The numeric or mnemonic of the routemaster destination associated with this device |
| &nbsp;&nbsp;&nbsp;&nbsp;`OperationalMode` | VALUE/UINT16 |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`CurrentUser` | VALUE/STRING |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Status` | VALUE/UINT16 |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`AvailableToUse` | VALUE/BOOL | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`InUse` | GROUP |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`Routed` | VALUE/BOOL | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`Reserved` | VALUE/BOOL | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`ForceInUse` | VALUE/BOOL |  |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`Timeline` | VALUE/STRING | R |  |  |  |  |
| &nbsp;&nbsp;&nbsp;&nbsp;`UserData` | GROUP |  |  |  |  |  |

## `04: Assignment Engine` — definition `Assignment Engine` (6 nodes)

| Path | Kind | Access | Units / range | Enum | Severities | Description |
|---|---|---|---|---|---|---|
| `Functions` | GROUP |  |  |  |  |  |
| &nbsp;&nbsp;`SAVE` | FUNCTION |  |  |  |  |  |
| &nbsp;&nbsp;`RELOAD` | FUNCTION |  |  |  |  |  |
| `EngineCount` | VALUE/UINT | R |  |  |  | The number of engines currently defined in the system |
| `RoutingMode` | TABLE |  |  |  |  |  |
| &nbsp;&nbsp;`Mode` | VALUE/INT64 | W | 0..2 |  |  |  |
