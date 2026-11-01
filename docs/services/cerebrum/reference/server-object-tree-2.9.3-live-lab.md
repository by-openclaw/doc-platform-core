# Cerebrum server — live walk of device `0.0.0.0` on the lab (2.9.3, 2026-10-22)

Read over the Northbound API from the platform controller with the `go-acp` connector (`dhs consumer cerebrum-nb tree --device 0.0.0.0 --sub-device N --path <top groups>`), one session per sub-device, start groups taken from the definition files ([`server-object-tree-2.9.3.md`](server-object-tree-2.9.3.md)). A standalone demo-licensed server with no Device Service, no clients and no redundancy: objects that need a partner answer `available=0`. Values are a snapshot, not a reference; the enum names are the wire's.

Do not edit by hand: regenerate from a new walk.

## `00: System` — 173 objects returned, 145 available

| Object | Value | Type | Enum | Available |
|---|---|---|---|---|
| `ComputerOverview.ProcessorTime` | 2 | INTEGER |  | yes |
| `ComputerOverview.ProcessCount` | 141 | INTEGER |  | yes |
| `ComputerOverview.ThreadCount` | 1511 | INTEGER |  | yes |
| `ComputerOverview.HandleCount` | 56501 | INTEGER |  | yes |
| `ComputerOverview.Memory.PhysicalInUse` | 6956 | INTEGER |  | yes |
| `ComputerOverview.Memory.PhysicalAvailable` | 58558 | INTEGER |  | yes |
| `ComputerOverview.Memory.CommittedTotal` | 6568 | INTEGER |  | yes |
| `ComputerOverview.Memory.CommittedPeak` | 8450 | INTEGER |  | yes |
| `ComputerOverview.Memory.CommittedLimit` | 80363 | INTEGER |  | yes |
| `ComputerOverview.Network.Transmit.BytesPerSec` | 4480 | INTEGER |  | yes |
| `ComputerOverview.Network.Transmit.BytesPerSecMax` | 561622 | INTEGER |  | yes |
| `ComputerOverview.Network.Transmit.PacketsPerSec` | 64 | INTEGER |  | yes |
| `ComputerOverview.Network.Transmit.PacketsPerSecMax` | 3669 | INTEGER |  | yes |
| `ComputerOverview.Network.Receive.BytesPerSec` | 5732 | INTEGER |  | yes |
| `ComputerOverview.Network.Receive.BytesPerSecMax` | 889170 | INTEGER |  | yes |
| `ComputerOverview.Network.Receive.PacketsPerSec` | 84 | INTEGER |  | yes |
| `ComputerOverview.Network.Receive.PacketsPerSecMax` | 5043 | INTEGER |  | yes |
| `ComputerOverview.Network.ResetPeakValues` |  | INTEGER |  | no |
| `ApplicationOverview.ProcessorTime` | 1 | INTEGER |  | yes |
| `ApplicationOverview.HandleCount` | 2975 | INTEGER |  | yes |
| `ApplicationOverview.PeakHandleCount` | 3964 | INTEGER |  | yes |
| `ApplicationOverview.GDIObjects` | 881 | INTEGER |  | yes |
| `ApplicationOverview.PeakGDIObjects` | 1649 | INTEGER |  | yes |
| `ApplicationOverview.UserObjects` | 1524 | INTEGER |  | yes |
| `ApplicationOverview.PeakUserObjects` | 1709 | INTEGER |  | yes |
| `ApplicationOverview.DeviceDeactivations` | 0 | INTEGER |  | yes |
| `ApplicationOverview.Memory.WorkingSet` | 824 | INTEGER |  | yes |
| `ApplicationOverview.Memory.PeakWorkingSet` | 888 | INTEGER |  | yes |
| `ApplicationOverview.Memory.CommitSize` | 764 | INTEGER |  | yes |
| `ApplicationOverview.Memory.PeakCommitSize` | 828 | INTEGER |  | yes |
| `InternalMemoryManagement.TotalReservedSize` |  | INTEGER |  | no |
| `InternalMemoryManagement.AllocatedBlocks` | 0 | STRING |  | yes |
| `InternalQueues.MessageQueueLength` | 0 | INTEGER |  | yes |
| `InternalQueues.PeakMessageQueueLength` | 6656 | INTEGER |  | yes |
| `InternalQueues.MessageQueue.PeakBacklogAge` | 17672 | INTEGER |  | yes |
| `InternalQueues.MessageQueue.PeakBacklogUser` | IP Rtr (0x0080) | STRING |  | yes |
| `InternalQueues.MessageQueue.CurrentBacklogAge` | 0 | INTEGER |  | yes |
| `InternalQueues.MessageQueue.CurrentBacklogUser` | None | STRING |  | yes |
| `InternalQueues.TotalMessageQueueLength` | 1022208 | INTEGER |  | yes |
| `InternalQueues.EventLogQueueLength` | 0 | INTEGER |  | yes |
| `InternalQueues.PeakEventLogQueueLength` | 0 | INTEGER |  | yes |
| `InternalQueues.TotalEventLogQueueLength` | 14592 | INTEGER |  | yes |
| `InternalQueues.LastEventLogCommitDuration` | 664 | INTEGER |  | yes |
| `InternalQueues.AuditTrailQueueLength` | 0 | INTEGER |  | yes |
| `InternalQueues.PeakAuditTrailQueueLength` |  | INTEGER |  | no |
| `InternalQueues.TotalAuditTrailQueueLength` | 2944 | INTEGER |  | yes |
| `InternalQueues.CorLinkSendQueue` | 0 | INTEGER |  | yes |
| `InternalQueues.PeakCorLinkSendQueue` |  | INTEGER |  | no |
| `InternalQueues.TotalCorLinkSendQueue` | 0 | INTEGER |  | yes |
| `InternalQueues.CorLinkReceiveQueue` | 0 | INTEGER |  | yes |
| `InternalQueues.PeakCorLinkReceiveQueue` |  | INTEGER |  | no |
| `InternalQueues.TotalCorLinkReceiveQueue` | 0 | INTEGER |  | yes |
| `InternalQueues.CorLinkRate` |  | INTEGER |  | no |
| `InternalQueues.ResetPeakQueueLengths` |  | INTEGER |  | no |
| `InternalQueues.WriteCount` | 8883 | INTEGER |  | yes |
| `ThreadResponsiveness.LastGUIDelay` | 0 | INTEGER |  | yes |
| `ThreadResponsiveness.PeakGUIDelay` | 0 | INTEGER |  | yes |
| `ThreadResponsiveness.MacroThreads` | 12 | STRING |  | yes |
| `Redundancy_Status.Is_Redundant_System` | 0 | INTEGER |  | yes |
| `Redundancy_Status.Overall` | OK | ENUM | Error\|OK | yes |
| `Redundancy_Status.Active_Server` | Primary | STRING |  | yes |
| `Redundancy_Status.Cerebrum_Partner` | Error | ENUM | Error\|OK | no |
| `Redundancy_Status.Cerebrum_Witness` | Error | ENUM | Error\|OK | no |
| `Redundancy_Status.SQL_Partner` | Error | ENUM | Error\|Synchronising\|OK | no |
| `Redundancy_Status.SQL_Witness` | Error | ENUM | Error\|OK | no |
| `Redundancy_Status.SQL_State` | Online | ENUM | Online\|Restoring\|Recovering\|RecoveryPending\|Suspect\|Emergency\|Offline | no |
| `Redundancy_Status.SQL_Log_Maintenance` | OK | ENUM | Error\|OK | yes |
| `Redundancy_Status.Watchdog_Health` | 100 | INTEGER |  | yes |
| `Federation.Is_Using_Federation_Service` | 0 | INTEGER |  | yes |
| `Federation.Overall` | Error | ENUM | Error\|OK | no |
| `Federation.Failover` |  | INTEGER |  | no |
| `Federation.Primary.Server_FQDN` |  | STRING |  | no |
| `Federation.Primary.Server_State` | Disconnected | STRING | Disconnected\|Inactive\|Deactivating\|Activating\|Active | no |
| `Federation.Primary.Server_Version` |  | STRING |  | no |
| `Federation.Secondary.Server_FQDN` |  | STRING |  | no |
| `Federation.Secondary.Server_State` | Disconnected | STRING | Disconnected\|Inactive\|Deactivating\|Activating\|Active | no |
| `Federation.Secondary.Server_Version` |  | STRING |  | no |
| `RouteMaster.TieLines.ConfigCount` | 0 | INTEGER |  | yes |
| `RouteMaster.TieLines.ActiveCount` | 0 | INTEGER |  | yes |
| `RouteMaster.TieLines.Shufflers.Rx_Defined` | 0 | INTEGER |  | yes |
| `RouteMaster.TieLines.Shufflers.Tx_Defined` | 0 | INTEGER |  | yes |
| `RouteMaster.TieLines.Shufflers.Tx_In_Use` | 0 | INTEGER |  | yes |
| `RouteMaster.TieLines.Errors.Count` | 0 | INTEGER |  | yes |
| `RouteMaster.TieLines.Errors.CountReset` |  | INTEGER |  | yes |
| `RouteMaster.TieLines.Errors.LastErrorMessage` |  | STRING |  | no |
| `ControlInterfaces.Name.[DHS].Connection` | 10.6.239.120:33232 | STRING |  | yes |
| `License.Sentinel_Key_Serial_Number` | Missing | STRING |  | yes |
| `License.License_Type` | Demo | STRING |  | yes |
| `License.Maintenance_Support_Expiry_Date` |  | STRING |  | no |
| `License.Demo_License_Enabled` | Enabled | ENUM | Disabled\|Enabled | yes |
| `License.Demo_License_Duration` | 10000 | STRING |  | yes |
| `License.Demo_License_Remaining` | 10000 | STRING |  | yes |
| `License.Supported_Options.Client_Licenses` | 10 | INTEGER |  | yes |
| `License.Supported_Options.Lite_Client_Licenses` | 0 | INTEGER |  | yes |
| `License.Supported_Options.Web_Client_Licenses` | 10 | INTEGER |  | yes |
| `License.Supported_Options.Panel_View` | Enabled | ENUM | Disabled\|Enabled | yes |
| `License.Supported_Options.Redundant_Server` | Allowed | ENUM | Allowed\|Compulsory | yes |
| `License.Supported_Options.Router_Control` | Enabled | ENUM | Disabled\|Enabled\|Enabled(Restart | yes |
| `License.Supported_Options.Route_Master` | Enabled | ENUM | Disabled\|Enabled | yes |
| `License.Supported_Options.Route_Master_Destination_Limit` | Unlimited | STRING |  | yes |
| `License.Supported_Options.Route_Master_Destinations_In_Use` | 15 | STRING |  | yes |
| `License.Supported_Options.IP_Destination_Limit` | 4096 | STRING |  | yes |
| `License.Supported_Options.IP_Destinations_In_Use` | 0 | STRING |  | yes |
| `License.Supported_Options.IP-Core` | Disabled | ENUM | Disabled\|Cerebrum | yes |
| `License.Supported_Options.IP_Shuffle_Limit` | 128 | INTEGER |  | yes |
| `License.Supported_Options.IP_Shuffle_In_Use` | 0 | INTEGER |  | yes |
| `License.Supported_Options.Max_IP_Destination_ID` | 4096 | STRING |  | yes |
| `License.Supported_Options.Tielines` | Enabled | ENUM | Disabled\|Enabled | yes |
| `License.Supported_Options.Salvos` | Enabled | ENUM | Disabled\|Enabled | yes |
| `License.Supported_Options.SNMP` | Enabled | ENUM | Disabled\|Enabled | yes |
| `License.Supported_Options.SNMP_Device_Limit` | 200 | STRING |  | yes |
| `License.Supported_Options.Tally_Manager` | Enabled | ENUM | Disabled\|Enabled | yes |
| `License.Supported_Options.Generic_Device_Points` | 500 | STRING |  | yes |
| `License.Supported_Options.Generic_Device_Points_In_Use` | 7 | STRING |  | yes |
| `License.Supported_Options.Northbound_API_Licenses` | 10 | INTEGER |  | yes |
| `License.Supported_Options.Northbound_API_Licenses_In_Use` | 0 | INTEGER |  | yes |
| `License.Supported_Options.Resource_Manager_Licenses` | 500 | INTEGER |  | yes |
| `License.Supported_Options.Resource_Manager_Licenses_In_Use` | 0 | INTEGER |  | yes |
| `License.Supported_Options.Assignment_Engine_Licenses` | 5 | INTEGER |  | yes |
| `License.Supported_Options.Assignment_Engine_Licenses_In_Use` | 2 | INTEGER |  | yes |
| `License.Supported_Options.Max_Package_Router` | 0 | INTEGER |  | yes |
| `License.Supported_Options.Package_Router_In_Use` | 0 | INTEGER |  | yes |
| `License.Supported_Options.External_Logging` | Disabled | ENUM | Disabled\|Enabled | yes |
| `License.Supported_Options.Federation` | 1 | INTEGER |  | yes |
| `ConfigDatabase.IndependentConfigState` | Not In Use | ENUM | Not | yes |
| `ConfigDatabase.PushToPartner` | 0 | INTEGER |  | yes |
| `ConfigDatabase.PullFromPartner` | 0 | INTEGER |  | yes |
| `ServerFileTransfer.PushToPartner` | 0 | INTEGER |  | yes |
| `ServerFileTransfer.Progress` |  | INTEGER |  | no |
| `SQLAlwaysOnStatus.GroupName` |  | STRING |  | no |
| `SQLAlwaysOnStatus.GroupId` |  | STRING |  | no |
| `Clients.LicencedCount` | 0 | INTEGER |  | yes |
| `Clients.LogOff` | 0 | INTEGER |  | yes |
| `Clients.PeakMessageQueueLength` | 0 | INTEGER |  | yes |
| `Clients.PeakMessageQueueAge` | 0 | INTEGER |  | yes |
| `Clients.ReceivedMessageCount` | 0 | INTEGER |  | yes |
| `Clients.ResetStatistics` | 0 | INTEGER |  | yes |
| `Clients.RichClientCount` | 0 | INTEGER |  | yes |
| `Clients.UpgradeClientToVersion` |  | STRING |  | yes |
| `Clients.RichClientStatus` | 1 | INTEGER |  | yes |
| `Clients.ChangeAccessControlList.IPAddress` |  | STRING |  | no |
| `Clients.ChangeAccessControlList.Status` | Unspecified | ENUM | Unspecified\|Allowed\|Denied | yes |
| `CorLinkStats.TX.DataPackets` | 0 | INTEGER |  | yes |
| `CorLinkStats.TX.Acks` | 0 | INTEGER |  | yes |
| `CorLinkStats.TX.Nacks` | 0 | INTEGER |  | yes |
| `CorLinkStats.TX.TimeOuts` | 0 | INTEGER |  | yes |
| `CorLinkStats.TX.Resends` | 1:0, 2:0, >0 | STRING |  | yes |
| `CorLinkStats.TX.Size` | 0 | INTEGER |  | yes |
| `CorLinkStats.TX.Compression` | 0 | FLOAT |  | yes |
| `CorLinkStats.RX.DataPackets` | 0 | INTEGER |  | yes |
| `CorLinkStats.RX.Acks` | 0 | INTEGER |  | yes |
| `CorLinkStats.RX.Nacks` | 0 | INTEGER |  | yes |
| `CorLinkStats.RX.Resends` | 0 | INTEGER |  | yes |
| `CorLinkStats.RX.Size` | 0 | INTEGER |  | yes |
| `CorLinkStats.RX.Compression` | 0 | FLOAT |  | yes |
| `CorLinkStats.ChangeRequestsReceived` | 0 | INTEGER |  | yes |
| `CorLinkStats.DataRequestsReceived` | 0 | INTEGER |  | yes |
| `CorLinkStats.ResetStats` | 0 | INTEGER |  | yes |
| `Users.User.[Admin].Group` | Admin | STRING |  | yes |
| `Users.User.[Admin].ControlCLF` |  | STRING |  | yes |
| `Users.User.[Admin].NavigationCLF` |  | STRING |  | yes |
| `Users.User.[Admin].UserType` | Cerebrum | ENUM | Cerebrum\|Active | yes |
| `Users.User.[dhs-staging].Group` | Admin | STRING |  | yes |
| `Users.User.[dhs-staging].ControlCLF` |  | STRING |  | yes |
| `Users.User.[dhs-staging].NavigationCLF` |  | STRING |  | yes |
| `Users.User.[dhs-staging].UserType` | Cerebrum | ENUM | Cerebrum\|Active | yes |
| `Foundation.Enabled` | 0 | INTEGER |  | yes |
| `Foundation.ExternalProxyFQDN` |  | STRING |  | yes |
| `Foundation.KeycloakAvailable` |  | INTEGER |  | no |
| `Foundation.KeycloakCertCurrent` |  | INTEGER |  | no |
| `Foundation.WebClientServiceIPAddresses` | 100 | STRING |  | yes |
| `Foundation.ConfigModified` | 0 | INTEGER |  | yes |
| `Foundation.RestartWebServer` | 0 | INTEGER |  | yes |

## `02: Services` — 7 objects returned, 7 available

| Object | Value | Type | Enum | Available |
|---|---|---|---|---|
| `ALL_RESPONSIVE` | OK | ENUM | Error\|OK | yes |
| `SERVICE_CREATION.SERVICE_NAME` |  | STRING |  | yes |
| `SERVICE_CREATION.SERVICE_TYPE` | Cerebrum Device Service | ENUM | Cerebrum | yes |
| `SERVICE_CREATION.SERVICE_LOCATION` |  | STRING |  | yes |
| `SERVICE_CREATION.CREATE` | 0 | INTEGER |  | yes |
| `ADD_SERVICE_MANAGER.IP_ADDRESS` | x.x.x.x | STRING |  | yes |
| `ADD_SERVICE_MANAGER.ADD` | 0 | INTEGER |  | yes |

## `03: Resource Manager` — 35 objects returned, 0 available

| Object | Value | Type | Enum | Available |
|---|---|---|---|---|
| `Functions.CreateAsset.AssetUID` |  | STRING |  | no |
| `Functions.DeleteAsset.AssetUID` |  | STRING |  | no |
| `Functions.CreateResource.AssetUID` |  | STRING |  | no |
| `Functions.CreateResource.ResourceUID` |  | STRING |  | no |
| `Functions.DeleteResource.AssetUID` |  | STRING |  | no |
| `Functions.DeleteResource.ResourceUID` |  | STRING |  | no |
| `Functions.FindResources.Available` |  | STRING |  | no |
| `Functions.FindResources.Start` |  | INTEGER |  | no |
| `Functions.FindResources.End` |  | INTEGER |  | no |
| `Functions.FindResources.FilterKey1` |  | STRING |  | no |
| `Functions.FindResources.FilterValue1` |  | STRING |  | no |
| `Functions.FindResources.FilterKey2` |  | STRING |  | no |
| `Functions.FindResources.FilterValue2` |  | STRING |  | no |
| `Functions.FindResources.FilterKey3` |  | STRING |  | no |
| `Functions.FindResources.FilterValue3` |  | STRING |  | no |
| `Functions.FindResources.FilterKey4` |  | STRING |  | no |
| `Functions.FindResources.FilterValue4` |  | STRING |  | no |
| `Functions.FindResources.FilterKey5` |  | STRING |  | no |
| `Functions.FindResources.FilterValue5` |  | STRING |  | no |
| `Functions.ReserveResource.AssetUID` |  | STRING |  | no |
| `Functions.ReserveResource.ResourceUID` |  | STRING |  | no |
| `Functions.ReserveResource.Start` |  | INTEGER |  | no |
| `Functions.ReserveResource.End` |  | INTEGER |  | no |
| `Functions.ReserveResource.Name` |  | STRING |  | no |
| `Functions.ReserveResource.Description` |  | STRING |  | no |
| `Functions.ModifyResourceReservation.AssetUID` |  | STRING |  | no |
| `Functions.ModifyResourceReservation.ResourceUID` |  | STRING |  | no |
| `Functions.ModifyResourceReservation.CurrentStart` |  | INTEGER |  | no |
| `Functions.ModifyResourceReservation.NewStart` |  | INTEGER |  | no |
| `Functions.ModifyResourceReservation.End` |  | INTEGER |  | no |
| `Functions.ModifyResourceReservation.Name` |  | STRING |  | no |
| `Functions.ModifyResourceReservation.Description` |  | STRING |  | no |
| `Functions.DeleteResourceReservation.AssetUID` |  | STRING |  | no |
| `Functions.DeleteResourceReservation.ResourceUID` |  | STRING |  | no |
| `Functions.DeleteResourceReservation.Start` |  | INTEGER |  | no |

## `04: Assignment Engine` — 10 objects returned, 6 available

| Object | Value | Type | Enum | Available |
|---|---|---|---|---|
| `Functions.SAVE.ENGINE` |  | STRING |  | no |
| `Functions.SAVE.SOURCE` |  | STRING |  | no |
| `Functions.SAVE.DESTINATION` |  | STRING |  | no |
| `Functions.RELOAD.ENGINE` |  | STRING |  | no |
| `EngineCount` | 5 | STRING |  | yes |
| `RoutingMode.[1].Mode` | 1-N | ENUM | 1-N\|1-1\|N-N | yes |
| `RoutingMode.[2].Mode` | 1-N | ENUM | 1-N\|1-1\|N-N | yes |
| `RoutingMode.[3].Mode` | 1-N | ENUM | 1-N\|1-1\|N-N | yes |
| `RoutingMode.[4].Mode` | 1-N | ENUM | 1-N\|1-1\|N-N | yes |
| `RoutingMode.[5].Mode` | 1-N | ENUM | 1-N\|1-1\|N-N | yes |

## `01: IP Router` — walk stopped at the 3 000-obtain cap: 8999 objects, all under `Senders` (600 sender rows, `Receivers` not reached)

The walk obtains every table row as a group (one request per sender), so the IP Router tables cost thousands of requests on a plant with the NMOS shuffler registered. For monitoring, subscribe to the table columns with the wire wildcard (`Senders.[*].Available`, `Receivers.[*].ReceiverStatus`) instead of walking.

Columns seen per sender row: `Device`, `UID`, `DeviceLabel`, `Label`, `ControlType`, `FromDevice`, `SignalType`, `Available`, `SenderStatus`, `Fed_By_Output`, `Stream_Status`, `NMS_ID_1`, `NMS_ID_2`, `FlowClashes`, `SubChannels`

Sample row:

| Object | Value |
|---|---|
| `Senders.[NMOS\|1\|lab-convert-01\|ATX-01].Device` | NMOS\|1 |
| `Senders.[NMOS\|1\|lab-convert-01\|ATX-01].UID` | lab-convert-01\|ATX-01 |
| `Senders.[NMOS\|1\|lab-convert-01\|ATX-01].DeviceLabel` | ATX-01 |
| `Senders.[NMOS\|1\|lab-convert-01\|ATX-01].Label` | ATX-01 |
| `Senders.[NMOS\|1\|lab-convert-01\|ATX-01].ControlType` | SDP |
| `Senders.[NMOS\|1\|lab-convert-01\|ATX-01].FromDevice` | 1 |
| `Senders.[NMOS\|1\|lab-convert-01\|ATX-01].SignalType` | audio/L24 |
| `Senders.[NMOS\|1\|lab-convert-01\|ATX-01].Available` | 1 |
| `Senders.[NMOS\|1\|lab-convert-01\|ATX-01].SenderStatus` | <Sender_Status TYPE=\"SDP\" CLF=\"Device/Generic Device/NMOS/IP_Rtr_Sender_Details_Popup.clf\"><EndP |
| `Senders.[NMOS\|1\|lab-convert-01\|ATX-01].Fed_By_Output` |  |
| `Senders.[NMOS\|1\|lab-convert-01\|ATX-01].Stream_Status` | 2 |
| `Senders.[NMOS\|1\|lab-convert-01\|ATX-01].NMS_ID_1` |  |
| `Senders.[NMOS\|1\|lab-convert-01\|ATX-01].NMS_ID_2` |  |
| `Senders.[NMOS\|1\|lab-convert-01\|ATX-01].FlowClashes` | 0 |
| `Senders.[NMOS\|1\|lab-convert-01\|ATX-01].SubChannels` | A1,A2,A3,A4,A5,A6,A7,A8,A9,A10,A11,A12,A13,A14,A15,A16 |
