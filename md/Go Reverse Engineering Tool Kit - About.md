> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [go-re.tk](https://go-re.tk/about/)

> A reverse engineering tool kit for Go binaries

The Go Reverse Engineering Tool Kit ([go-re.tk](https://go-re.tk/)) is a new open-source toolset for analyzing Go binaries. The tool is designed to extract as much metadata as possible from stripped binaries to assist in both reverse engineering and malware analysis. For example, GoRE can detect the compiler version used, extract type information, and recover function information, including source code line numbers for functions and source tree structure.

The core library is written in Go, but the tool kit includes C-bindings and a library implementation in Python. When using the C-bindings or the Python library, it is possible to write plugins for other analysis tools such as IDA Pro and Ghidra. The toolset also includes “redress”, which is a command line tool to “dress” stripped Go binaries. It can both be used standalone to print out extracted information from the binary or as a radare2 plugin to reconstruct stripped symbols and type information.

The tool kit consists of:

*   Core library written in Go
*   C-bindings
*   Python library using the C-bindings
*   A [command line tool](https://go-re.tk/redress) for easy analysis

Function information recovery
-----------------------------

Sample output of the command line tool below prints a representation of the source code structure for one of the Zebrocy backdoor and downloader samples (672f85ee2031c1ff6295b17058e30e07). The output includes the filename of the source files, function names, function’s starting line number and ending line number in the file, and function line number length.

```
Package main: C:/!Project/C1/ProjectC1Dec

File: main.go
        GetDisk Lines: 27 to 37 (10)
        Getfilename Lines: 37 to 52 (15)
        CMDRunAndExit Lines: 52 to 58 (6)
        Tasklist Lines: 58 to 70 (12)
        Installed Lines: 70 to 82 (12)
        Session_List Lines: 82 to 97 (15)
        List_Local_Users Lines: 97 to 119 (22)
        systeminformation Lines: 119 to 124 (5)
        CreateSysinfo Lines: 124 to 137 (13)
        ParseData Lines: 137 to 153 (16)
        SendPOSTRequests Lines: 153 to 174 (21)
        Screen Lines: 174 to 192 (18)
        GetSND Lines: 192 to 215 (23)
        PrcName Lines: 215 to 222 (7)
        main Lines: 222 to 229 (7)


```

Type information recovery
-------------------------

The sample output below is data struct extracted from a DDGS botnet sample (8c2e1719192caa4025ed978b132988d6). Structure field names and method names with first character lowercase are private fields and hence not accessible outside the package. Still, the tool kit can extract this information.

```
type main.Payload struct{
        CfgVer int
        Cmds cmd.Table
        Miners []main.miner
}

type main.miner struct{
        cmd.XRun
}

type xlist.XList struct{
        *memberlist.Memberlist
}
func (xlist.XList) GetHealthScore() int
func (xlist.XList) Join([]string) (int, error)
func (xlist.XList) Leave(int64) error
func (xlist.XList) LocalNode() *memberlist.Node
func (xlist.XList) Members() []*memberlist.Node
func (xlist.XList) NumMembers() int
func (xlist.XList) Ping(string, net.Addr) (int64, error)
func (xlist.XList) ProtocolVersion() uint8
func (xlist.XList) SendBestEffort(*memberlist.Node, []uint8) error
func (xlist.XList) SendReliable(*memberlist.Node, []uint8) error
func (xlist.XList) SendTo(net.Addr, []uint8) error
func (xlist.XList) SendToTCP(*memberlist.Node, []uint8) error
func (xlist.XList) SendToUDP(*memberlist.Node, []uint8) error
func (xlist.XList) UpdateNode(int64) error
func (xlist.XList) aliveNode()
func (xlist.XList) anyAlive() bool
func (xlist.XList) deadNode()
func (xlist.XList) decryptRemoteState()
func (xlist.XList) deschedule()
func (xlist.XList) encodeAndBroadcast()
func (xlist.XList) encodeAndSendMsg()
func (xlist.XList) encodeBroadcastNotify()
func (xlist.XList) encryptLocalState([]uint8) ([]uint8, error)
func (xlist.XList) encryptionVersion()
func (xlist.XList) estNumNodes() int
func (xlist.XList) getBroadcasts(int, int) [][]uint8
func (xlist.XList) gossip()
func (xlist.XList) handleAck()
func (xlist.XList) handleAlive()
func (xlist.XList) handleCommand()
func (xlist.XList) handleCompound()
func (xlist.XList) handleCompressed()
func (xlist.XList) handleConn()
func (xlist.XList) handleDead()
func (xlist.XList) handleIndirectPing()
func (xlist.XList) handleNack()
func (xlist.XList) handlePing()
func (xlist.XList) handleSuspect()
func (xlist.XList) handleUser()
func (xlist.XList) ingestPacket()
func (xlist.XList) invokeAckHandler()
func (xlist.XList) invokeNackHandler()
func (xlist.XList) mergeRemoteState()
func (xlist.XList) mergeState()
func (xlist.XList) nextIncarnation() uint32
func (xlist.XList) nextSeqNo() uint32
func (xlist.XList) packetHandler()
func (xlist.XList) packetListen()
func (xlist.XList) probe()
func (xlist.XList) probeNode()
func (xlist.XList) pushPull()
func (xlist.XList) pushPullNode(string, bool) error
func (xlist.XList) pushPullTrigger()
func (xlist.XList) queueBroadcast()
func (xlist.XList) rawSendMsgPacket()
func (xlist.XList) rawSendMsgStream()
func (xlist.XList) readRemoteState()
func (xlist.XList) readStream()
func (xlist.XList) readUserMsg()
func (xlist.XList) refute()
func (xlist.XList) resetNodes()
func (xlist.XList) resolveAddr()
func (xlist.XList) schedule()
func (xlist.XList) sendAndReceiveState()
func (xlist.XList) sendLocalState()
func (xlist.XList) sendMsg()
func (xlist.XList) sendPingAndWaitForAck()
func (xlist.XList) sendUserMsg()
func (xlist.XList) setAckHandler()
func (xlist.XList) setAlive() error
func (xlist.XList) setProbeChannels()
func (xlist.XList) skipIncarnation()
func (xlist.XList) streamListen()
func (xlist.XList) suspectNode()
func (xlist.XList) tcpLookupIP()
func (xlist.XList) triggerFunc()
func (xlist.XList) verifyProtocol()


```