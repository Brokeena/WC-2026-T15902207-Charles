
## 1. Goal

In the previous labs, you analyzed 5G signaling messages and network performance using provided captures and measurements.

In this lab, you will deploy your own 5G Standalone (SA) network using OpenAirInterface (OAI), verify the complete connection, investigate RFsimulator communication, perform active network measurements, and troubleshoot a controlled connection failure.

Unlike the previous labs, you will collect measurements directly from your own running 5G network.

**Learning Objectives**

After completing this lab, you should be able to:

1. Deploy an OAI gNB, nrUE, and 5G Core Network.
2. Identify the main stages of 5G SA connection establishment.
3. Verify communication between gNB and nrUE through RFsimulator.
4. Measure user-plane connectivity under different packet sizes.
5. Diagnose connection failures using OAI and 5GC logs.
6. Explain the limitations of a software-simulated 5G network.
****
## 2. Experimental Environment

Use the *OAI v2.4.0 Build Guide* to prepare the environment before class.

| Component | Configuration |
|---|---|
| Ubuntu | 24.04 |
| OAI RAN | v2.4.0 |
| 5G Core | OAI CN5G |
| gNB | nr-softmodem |
| UE | nr-uesoftmodem |
| Radio interface | RFsimulator |
| Band | n78 |
| Subcarrier spacing | 30 kHz |
| Resource blocks | 106 |
| UE IMSI | 001010000000001 |
| External DN | 192.168.70.135 |

The experiment uses one Ubuntu machine without physical RF hardware.

## 3. Task 1 — Deploy and Verify the Complete 5G SA Network

Start the OAI CN5G:

```bash
cd ~/oai-cn5g
docker compose up -d
docker compose ps
```

Start the gNB and nrUE using the commands in the Build Guide.

Observe their terminal outputs and locate evidence of the following stages.

| Stage                        | Successful? | Evidence from logs                                                                                                                                                                                                                                             |
| ---------------------------- | ----------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| CN5G initialization          | Yes         | ![](lab3-files/Pasted%20image%2020261111123555.png)<br><br>Relevant logs :<br>*CN5G container startup*                                                                                                                                                         |
| gNB NG Setup                 | Yes         | ![](lab3-files/Pasted%20image%2020261111124003.png)<br>*Successful gNB NG Setup with the AMF*<br><br>Relevant logs :<br>`[NGAP] Received NGSetupResponse from AMF`                                                                                             |
| UE synchronization           | Yes         | ![](lab3-files/Pasted%20image%2020261111123905.png)<br>*Successful UE synchronization with the gNB*<br><br>Relevant logs :<br>`Initial sync successful, PCI: 0`                                                                                                |
| RRC connection establishment | Yes         | ![](lab3-files/Pasted%20image%2020261111124433.png)<br>*UE RRC connection establishment*<br><br>Relevant logs :<br>`State = NR_RRC_CONNECTED`                                                                                                                  |
| UE Registration              | Yes         | ![](lab3-files/Pasted%20image%2020261111124818.png)<br>*UE Registration Accept*<br><br>Relevant logs :<br>```[NAS] [UE 0] Received NAS_DOWNLINK_DATA_IND type FGS_REGISTRATION_ACCEPT with length 53<br>[NAS] Received Registration Accept with result 3GPP``` |
| PDU Session Establishment    | Yes         | ![](lab3-files/Pasted%20image%2020261111125051.png)<br>*PDU Session Establishment Accept*<br><br>Relevant logs :<br>`[NAS] Send NAS_UPLINK_DATA_REQ message(PduSessionEstablishRequest)`                                                                       |
| UE tunnel creation           | Yes         | ![](lab3-files/Pasted%20image%2020261111125149.png)<br>*UE tunnel creation with IPv4 address 10.0.0.2*<br><br>Relevant logs :<br>`Interface oaitun_ue1 successfully configured, IPv4 10.0.0.2`                                                                 |

Take screenshots showing the gNB and UE successfully connected.
![](lab3-files/Pasted%20image%2020261111125524.png)
*Successful UE Registration, PDU Session Establishment and tunnel creation.*

```console 
[NAS] Received Registration Accept with result 3GPP
[NAS] Send NAS_UPLINK_DATA_REQ message(RegistrationComplete)
[NAS] Received PDU Session Establishment Accept, UE IPv4: 10.0.0.2
[OIP] Interface oaitun_ue1 successfully configured, IPv4 10.0.0.2
```
*Relevant logs*

**Questions:**

1. Does a successful NG Setup guarantee that a UE can register? Explain.
> No. NG Setup only confirms that the gNB is connected to the AMF. UE Registration still requires successful authentication and signaling procedures.
2. Why can the UE establish an RRC connection but still fail Registration?
> RRC establishes the radio connection between the UE and gNB, while Registration requires authentication by the 5G Core Network. Registration can fail due to an incorrect IMSI or authentication failure.
3. What additional procedure is required after Registration before user data can be exchanged?
> The UE must complete PDU Session Establishment, which enables user-plane connectivity and assigns an IP address to the UE.

## 4. Task 2 — Investigate RFsimulator Communication

OAI RFsimulator replaces the physical RF interface with software-based communication.

The gNB and nrUE exchange IQ samples through a TCP connection.

Check the running processes:

```bash
pgrep -a nr-softmodem
pgrep -a nr-uesoftmodem
```

Inspect the TCP connections:

```bash
sudo ss -tnp | grep 4043
```

If necessary, inspect the listener:

```bash
sudo ss -ltnp | grep 4043
```

The default RFsimulator port in this experiment is 4043.
![](lab3-files/Pasted%20image%2020261111130825.png)
*Verification of the Established RFsimulator TCP Connection Between gNB and nrUE*

Complete the table:

| Item                           | Observed Value                                           |
| ------------------------------ | -------------------------------------------------------- |
| gNB process ID                 | 10199, 10200                                             |
| nrUE process ID                | 34875, 34876                                             |
| RFsimulator transport protocol | TCP                                                      |
| RFsimulator port               | 4043                                                     |
| TCP connection state           | `ESTAB` (Established)                                    |
| Connected process(es)          | `nr-softmodem` (PID 10199), `nr-uesoftmodem` (PID 34875) |

Take one screenshot showing the TCP connection.

**Questions:**

1. Why does RFsimulator use a TCP connection instead of a physical antenna?
> RFsimulator uses TCP to exchange simulated IQ samples between the gNB and UE, allowing wireless communication to be tested without physical RF hardware.
2. Are the TCP packets exchanged by RFsimulator the same as the UE's application-layer IP packets?
> No. RFsimulator TCP packets transport IQ samples representing radio signals, whereas the UE's application-layer IP packets travel through the established 5G user plane.
3. Does using RFsimulator mean that the UE and gNB are performing no physical-layer processing? Explain.
> No. The UE and gNB still perform physical-layer processing, including OFDM modulation and demodulation. RFsimulator replaces the physical radio interface, not the entire PHY layer.

## 5. Task 3 — Verify End-to-End Connectivity

Check the UE tunnel interface:

```bash
ip -4 addr show oaitun_ue1
```

Test connectivity to the External DN:

```bash
ping -I oaitun_ue1 -c 10 192.168.70.135
```

Record:

| Measurement         | Observed Value |
| ------------------- | -------------- |
| UE IP address       | 10.0.0.2       |
| Destination IP      | 192.168.70.135 |
| Packets transmitted | 10             |
| Packets received    | 10             |
| Packet loss (%)     | 0              |
| Minimum RTT (ms)    | 7.874 ms       |
| Average RTT (ms)    | 11.669 ms      |
| Maximum RTT (ms)    | 17.072 ms      |

Take a screenshot of the Ping result.
![](lab3-files/Pasted%20image%2020261111131656.png)
*Connectivity test*

**Questions:**

1. Why must Ping use the `oaitun_ue1` interface in this experiment?
> Because oaitun_ue1 is the simulated UE's tunnel interface. Using it ensures that the ICMP packets travel through the 5G user plane instead of the VM's normal network interface.
2. Does successful Registration guarantee successful user-plane connectivity?
> No. Registration only confirms that the UE is registered with the 5G Core. A successful PDU session and correctly configured userplane routing are also required.
3. What does a successful Ping prove, and what does it not prove about the simulated 5G system?
> A successful ping proves that IP packets can travel between the UE and destination through the simulated 5G network. However, it does not prove that the network would achieve the same latency, reliability or performance over a real wireless channel.

## 6. Task 4 — Investigate the Effect of Packet Size

You will now investigate whether changing the ICMP packet size affects the observed RTT.

Perform three experiments using the same established 5G connection.

**Scenario A — Small Packets**

```bash
ping -I oaitun_ue1 -s 56 -c 30 -i 0.2 192.168.70.135
```

![](lab3-files/Pasted%20image%2020261111132516.png)

*Scenario A: Ping results with a 56-byte ICMP payload*

**Scenario B — Medium Packets**

```bash
ping -I oaitun_ue1 -s 512 -c 30 -i 0.2 192.168.70.135
```

![](lab3-files/Pasted%20image%2020261111132549.png)

*Scenario B: Ping results with a 512-byte ICMP payload*

**Scenario C — Large Packets**

```bash
ping -I oaitun_ue1 -s 1200 -c 30 -i 0.2 192.168.70.135
```

![](lab3-files/Pasted%20image%2020261111132632.png)

*Scenario C: Ping results with a 1200-byte ICMP payload*

Note: `-s` specifies ICMP payload size, not the complete IP packet size.

Complete the table:

| Scenario | ICMP Payload | Avg. RTT (ms) | Max. RTT (ms) | Packet Loss (%) |
| -------- | -----------: | ------------: | ------------: | --------------: |
| A        |     56 bytes |        10.402 |        13.461 |               0 |
| B        |    512 bytes |        11.037 |        13.791 |               0 |
| C        |   1200 bytes |        12.874 |        16.892 |               0 |

Calculate the RTT difference:

```text
RTT Difference = Average RTT(C) - Average RTT(A)
			   = 12.874 - 10.402
			   = 2.472 ms
```

**Questions:**
1. Which scenario has the highest average RTT?
> Scenario C (1200 bytes) has the highest average RTT of 12.874 ms, compared to 11.037 ms for Scenario B and 10.402 ms for Scenario A.
2. Does increasing packet size always increase RTT in your measurements?
> Yes. In our measurements, the average RTT increased with packet size. However, this is not always guaranteed because other factors can affect network latency.
3. Why might a larger packet require more transmission or processing resources?
> Larger packets contain more data, requiring additional transmission time and processing resources for encoding, modulation, and decoding.
4. What factors other than packet size could affect RTT in a software-based 5G network?
> RTT can be affected by CPU load, operating system scheduling, memory usage, and network buffering.
5. Can these results be interpreted as real over-the-air 5G latency? Explain.
> No. RFsimulator replaces physical radio transmission with software-based communication. Therefore, the measurements do not account for real-world factors such as interference, signal attenuation, and wireless propagation conditions.

Do not assume that RTT must increase with packet size. Use your measured values to support your explanation.

## 7. Task 5 — Diagnose a Controlled UE Registration Failure

In this task, you will deliberately introduce one configuration error and investigate why the UE cannot complete registration.

**Important:** Save the original successful UE command before starting. Do not modify the CN5G subscriber database.

### Step 1 — Stop the UE

Press `Ctrl+C` in the nrUE terminal. Keep the CN5G and gNB running.

### Step 2 — Launch the UE with an Incorrect IMSI

Change only the IMSI:

```bash
sudo ./nr-uesoftmodem \
  -r 106 \
  --numerology 1 \
  --band 78 \
  -C 3619200000 \
  --uicc0.imsi 001010000000099 \
  --rfsim \
  --rfsimulator.serveraddr 127.0.0.1
```

The IMSI `001010000000099` is not provisioned in the standard classroom subscriber configuration. Verify this assumption against the instructor's database before class.

![](lab3-files/Pasted%20image%2020261111134733.png)
*Incorrect IMSI UE launch*

```console
[SIM] UICC simulation: IMSI=001010000000099
```
*Relevant logs*

![](lab3-files/Pasted%20image%2020261111134855.png)
*Incorrect IMSI UE launch*

```console
[PHY] Initial sync successful, PCI: 0
[PHY] UE synchronized!
[NR_MAC] 4-Step RA procedure succeeded.
[NR_RRC] State = NR_RRC_CONNECTED
[NAS] Generate Initial NAS Message: Registration Request
[NAS] [UE 0] Received NAS_DOWNLINK_DATA_IND type FGS_REGISTRATION_REJECT with length 4
[NAS] Received Registration reject message too short
```
*Relevant logs*

Observe the UE and AMF logs.

```bash
cd ~/oai-cn5g
docker compose logs --since=2m oai-amf
```

![](lab3-files/Pasted%20image%2020261111134340.png)
*UE/AMF logs*

```console
[06:39:16.633] Got UE Authentication from AUSF:
{"cause":"CONTEXT_NOT_FOUND","detail":"Resource corresponding to User imsi-001010000000099 not found","status":404}

[06:39:16.637] Request Authentication Vectors failure

[06:39:16.637] 5GMM state transition: COMM-PROC-INIT -> 5GMM-DEREGISTERED (event: [REGISTRATION_REJECT_SENT])

[06:39:16.638] Initiating RegistrationReject
```
*Relevant logs*

Allow time for registration attempts, then stop the UE.

Complete the table:

| Item                        | Observation                                                    |
| --------------------------- | -------------------------------------------------------------- |
| Modified parameter          | IMSI changed from 001010000000001 to 001010000000099           |
| UE synchronized with gNB?   | Yes                                                            |
| RRC connection established? | Yes                                                            |
| Registration successful?    | No                                                             |
| Error or rejection evidence | `FGS_REGISTRATION_REJECT`; AUSF `CONTEXT_NOT_FOUND` (HTTP 404) |
| Likely cause                | Incorrect IMSI not found in the subscriber database            |

If the expected registration rejection does not appear, report the actual logs instead of inventing a failure message.

### Step 3 — Restore the Original IMSI

Restart the nrUE using:

```text
--uicc0.imsi 001010000000001
```

Confirm that the UE successfully registers again and that Ping works.
![](lab3-files/Pasted%20image%2020261115164520.png)
*Successful Ping through oaitun_ue1 after restoring the original IMSI*


**Questions:**

1. Why can the UE potentially synchronize with the gNB even when the IMSI is incorrect?
> UE synchronization relies on physical-layer signals transmitted by the gNB, not on the IMSI. Therefore, the UE can synchronize and establish an RRC connection even with an incorrect IMSI.
2. Which network function is involved in UE authentication?
> The AUSF (Authentication Server Function) handles UE authentication, working with the UDM. In our experiment, the AUSF returned CONTEXT_NOT_FOUND (HTTP 404) because the subscriber was not found.
3. Why does changing an IMSI affect Registration but not the RFsimulator TCP connection itself?
> The IMSI is used for subscriber identification and authentication during Registration. RFsimulator uses TCP to exchange IQ samples independently of the UE's subscriber identity.
4. What evidence proves that restoring the correct configuration solved the problem?
> After restoring the original IMSI (`001010000000001`), the UE successfully exchanged 5 ICMP packets with the external data network, achieving 0% packet loss and an average RTT of 10.885 ms. This confirms that end-to-end user-plane connectivity was restored.

Submit evidence from both the failed and recovered connections.


## 8. Task 7 — Explain the Relationship Between Simulation and Real 5G

Your OAI gNB and nrUE perform 5G NR protocol processing even though they do not use physical antennas.

Complete the comparison:

| Feature                      | OAI RFsimulator | Physical 5G System |
| ---------------------------- | --------------- | ------------------ |
| gNB and UE protocol stack    | Yes             | Yes                |
| OFDM signal processing       | Yes             | Yes                |
| Physical antennas            | No              | Yes                |
| RF hardware impairments      | Not inherently  | Yes                |
| Wireless propagation effects | Not inherently  | Yes                |
| 5G Core connectivity         | Yes             | Yes                |

**Questions:**

1. Which parts of the 5G system remain functional when physical RF hardware is replaced with RFsimulator?
> The 5G protocol stack remains functional, including PHY, MAC, RLC, PDCP, RRC, and NAS. OFDM processing, UE Registration, PDU Session Establishment, and 5G Core connectivity are also supported.
2. Why is RFsimulator useful for testing the 5G protocol stack?
> RFsimulator allows developers to test and debug 5G communication without expensive radio hardware. It provides a controlled and reproducible environment for analyzing network procedures.
3. Why might latency and throughput differ between RFsimulator and a real 5G deployment?
> RFsimulator performance depends on CPU processing, operating system scheduling, and virtualization overhead. Real 5G networks are also affected by signal attenuation, interference, fading, and wireless propagation conditions.
4. What would need to change if you wanted to test a physical UE instead of OAI nrUE?
> The simulated RF interface would need to be replaced with compatible RF hardware, such as a USRP and antennas. A physical 5G device with a properly configured SIM would also be required, along with authorized radio spectrum for transmission.

## 9. Final Analysis

Answer each question in three to five sentences.

1. Describe the complete process from starting the OAI 5GC to successful user-plane connectivity.
> First, the 5G Core Network is started using Docker, followed by the gNB, which establishes an NG connection with the AMF. The UE then synchronizes with the gNB, establishes an RRC connection, and completes Registration. After successful PDU Session Establishment, the UE receives the IP address 10.0.0.2 through the oaitun_ue1 interface. Finally, successful Ping tests with 0% packet loss confirm end-to-end user-plane connectivity.
2. Explain the difference between an RFsimulator TCP connection and a 5G PDU Session.
> RFsimulator uses a TCP connection on port 4043 to exchange IQ samples between the gNB and UE, replacing the physical radio interface. In contrast, a 5G PDU Session provides user-plane connectivity between the UE and the data network through the 5G Core. Therefore, RFsimulator transports simulated radio signals, while the PDU Session enables the transmission of application data and IP packets.
3. Based on your measurements, how did packet size affect RTT and packet loss?
> The average RTT increased with packet size, from 10.402 ms for 56-byte packets to 11.037 ms for 512-byte packets and 12.874 ms for 1200-byte packets. The difference between the smallest and largest packets was 2.472 ms. All three scenarios showed 0% packet loss, indicating reliable communication during the experiments. However, virtualization and CPU scheduling may also have influenced the measured RTT.
4. Describe the registration failure you introduced and explain how you diagnosed and recovered from it.
> The UE's IMSI was changed from 001010000000001 to 001010000000099, causing a Registration failure. Although synchronization and RRC connection establishment succeeded, the AUSF returned an HTTP 404 error (`CONTEXT_NOT_FOUND`) because the subscriber was not found. The AMF subsequently initiated a Registration Reject. After restoring the original IMSI, a Ping test confirmed successful connectivity with 5 packets received, 0% packet loss, and an average RTT of 10.885 ms.
5. Identify two limitations of this simulated environment and explain how they might affect experimental results.
> The first limitation is the absence of real wireless propagation effects, such as interference, fading, and signal attenuation, which can significantly affect 5G performance. The second limitation is the dependence on software processing and virtualization, where CPU load and operating system scheduling can influence latency and throughput. Consequently, measurements obtained using RFsimulator may differ from those of a real 5G deployment. These limitations mean that simulated results cannot be directly interpreted as real-world wireless performance.

## 10. Submission Checklist

Submit one Markdown or PDF report containing:

- Completed 5G SA connection verification table.
- gNB and nrUE connection screenshots.
- RFsimulator TCP connection evidence.
- Baseline Ping measurement.
- Three packet-size comparison results.
- RTT difference calculation.
- Controlled failure and recovery evidence.
- RFsimulator versus physical 5G comparison.
- Answers to the final analysis questions.

All measurements and log evidence must come from your own experimental environment.

## 11. Grading Rubric

| Item | Points |
|---|---:|
| Complete 5G SA deployment | 15 |
| RFsimulator TCP connection analysis | 15 |
| End-to-end user-plane verification | 15 |
| Packet-size experiment and analysis | 20 |
| Controlled failure and recovery | 20 |
| Simulation limitations and final analysis | 15 |
| **Total** | **100** |

Vote TOP3：https://forms.gle/Mvza68DFBzPcZXfW6
