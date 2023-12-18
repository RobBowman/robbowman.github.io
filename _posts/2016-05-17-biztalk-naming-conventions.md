---
title: BizTalk Naming Conventions
category: BizTalk
tags:
    - BizTalk
---
# BizTalk Naming Conventions
Naming of BizTalk stuff is no different than for any programming language. It's helpful if multiple developers on the same project use consistent terms. I've had a preference to how things like receive and send ports are named for years but I've never written them down - until now!

This list is a work-in-progress, I intend adding too it as time allows.


<table>
<thead>
<tr>
<th>Artefact</th>
<th>Convention</th>
<th>Example</th>
<th>Notes</th>
</tr>
<thead>
<tbody>
<tr>
<td>Physical Ports</td>
</tr>
<tr>
<td>Physical Receive Port</td>
<td><i>Rcv</i>_TypeFromSource</td>
<td><i>Rcv</i>_EmployeeUpdateFromSystemX</td>
<td>The port direction (Rcv or Send) is separated from the purpose by an underscore to aid readability and ensure the direction is immediately obvious</td>
</tr>
<tr>
<td>Physical Receive Location</td>
<td><i>Rcv</i>_TypeFromSource_TRANSPORT</td>
<td><i>Rcv</i>_EmployeeUpdateFromSystemX_FILE</td>
<td>The adapter transport is given in upper case as a highlight but also because often a receive port may have multiple locations that differ only by the transport that is used</td>
</tr>
<tr>
<td>Physical Send Port (One Way)</td>
<td><i>Send</i>_TypeToDestination_TRANSPORT</td>
<td>Send_EmployeeUpdateToSystemY_WebHttp</td>
<td>The adapter transport is given in upper case as a highlight but also because often multiple send ports may exist that differ only by the adapter that is used</td>
</tr>
<tr>
<td>Physical Send Port (Two Way)</td>
<td><i>Request</i>_TypeToDestination_TRANSPORT</td>
<td>Request_EmployeeUpdateToSystemY_WebHttp</td>
<td>I like to use the 'Request' prefix for two-way send ports otherwise you can end up with names like Send_GetCustomerId</td>
</tr>
<tr>
<td>BTDF Environment Settings</td>
</tr>
<tr>
<td>Send File Path</td>
<td><i>SendPort</i>.Address</td>
<td>Send_EmployeeUpdateToSystemY_FILE.Address</td>
<td>A full stop is used to separate port name with its properties. The value from the excel spreadsheet gets substituted into PortBindings e.g <Address>${Send_HR_FILE.Address}</Address></td>
</tr>
<tr>
<td>Receive File Path</td>
<td><i>ReceiveLocation</i>.Address</td>
<td>Rcv_EmployeeUpdateFromSystemX_FILE.Address</td>
<td></td>
</tr>
</tbody>
</table>