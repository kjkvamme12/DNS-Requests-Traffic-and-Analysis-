# DNS Requests, Traffic, and Analysis

## Objective



### Skills Learned

- Manually generate DNS requests of different types from the command line to become familiar with results.
- Record DNS requests and observe how it will appear in Wireshark.
- Learn hwo to find DNS records and subdomains, without creating any traffic that could alert attackers.
- View DoH in a PCAP and see how it differs from traditional DNS, and also how it is the same.
- Analyze an incident.

### Tools Used
- Wireshark 


## Steps


## Generate DNS requests with dig and record them

1. Enter a command that will run tcpdump and have it record on port 53, do not resolve names for IP address, and listen on interface ens33 (our network interface) , write everything to a file called dns.pcap

![Screenshot 2025-01-28 171201](https://github.com/user-attachments/assets/670fc262-cf5b-4a86-a0ed-e3edd21732b9)

### 2. Send an A record request for sec450.com

   
![Screenshot 2025-01-28 171549](https://github.com/user-attachments/assets/47125385-39f1-471e-9cde-25b6687d8e9a)

The IP address at the time was 70.32.97.206
-QUESTION section shows that we made an A record request about the hostname sec450.com
- ANSWER section shows the returned answer
- SERVER line shows the DNS server tha answered the request (local stub resolver in this case)

### 2. Perform a PTR record lookup.
We will take the IP address found from the previous A record request for sec450.com

   
![Screenshot 2025-01-28 171928](https://github.com/user-attachments/assets/2ef6541d-98da-44ed-9b24-809691e78ca7)

The answer is not sec450. This is because the IP address where the sec450.com A record points is a shared server used for multiple purposes. Because of this, the PTR record no longer gives useful information about all the domains that may point to the IP with the A record. 
- The owner of the IP address will get to set the PTR records, but the owner of a domain name sets the A records to point to an IP address.
- PTR records can be pointed anywhere

### 3. TXT Record Request


![Screenshot 2025-01-28 172545](https://github.com/user-attachments/assets/372cd865-e313-4efe-a283-17249cf28808)


### 4. NS Record Request 


![Screenshot 2025-01-28 174207](https://github.com/user-attachments/assets/07a1283d-e8b9-4572-b5cd-732db1ff1886)

You can see at the bottom in the ANSWER SECTION the hosts for the sec450.com DNS. These are the authoritative nameservers that we would use. 

### Stop your tcpdump recording
We now will stop our tcpdump recording. Once we do so it will have captured the DNS traffic into a file called dns.pcap

## View PCAPs for the DNS traffic in Wireshark

### 1. Filter through Wireshark looking for just DNS traffic.

   

![Screenshot 2025-01-28 174710](https://github.com/user-attachments/assets/0696b630-f4ab-41c9-a9ae-2e8fe6273939)

### 2. Building filters in wireshark 

- Craft a filter for A record lookups only.
  1. CLick on the A record lookup in the capture
  2. Look at the second panel in Wireshark and expand the lowermost line that says Dmonain Name System (query).
  3. Click on Queries line and unfold that.
  4. Click where it says sec450.com: type A, class IN
  5. then select Type: A (Host Address) (1)
 

![Screenshot 2025-01-28 175109](https://github.com/user-attachments/assets/54cd22b8-c1fe-44a8-9be7-64dbe6ac0316)

 This is the line that singles out the request query type. 
We now will right-click on the line and select "Apply as filter" then Selected

![Screenshot 2025-01-28 175312](https://github.com/user-attachments/assets/11fd868b-80f6-4b52-a9fd-cb7014ff0a54)

Now Wireshark has applied the following filter in the search bar: "dns.qry.type == 1"


![Screenshot 2025-01-28 175447](https://github.com/user-attachments/assets/03dfe780-43a9-4ed8-8768-9b665b79742f)

Wireshark now has applied a new filter. To break it down: 
- dns.qry.name == "sec450.com"   this states that the request must be for this exact domain name. If you wanted to filter down to only requests for given hostname this format can be used
- &&   this is an AND logical operator in Wireshark
- dns.qr.type == 1  This means filter this down to type 1 queries

A type 1 query show A record requests only. 

### Making it easier to read queries

The apply as column functionality. 

![Screenshot 2025-01-28 175947](https://github.com/user-attachments/assets/82c9c138-c4ca-445b-94f2-6b53a2a84075)


Now there is a column that will cut out just the domain name from the request. 

### Passive DNS Data and Open-Source Intelligence

Passive DNS data sources should be your first move when resolving a hostname to an IP. This is because you can do so without actually having to send any DNS packets. 
We can get passive DNS data from existing logs, or open source intelligence. A plus side to open-source intelligence is that it gives you list of known subdomains as well. 

### Analyzing DNS over HTTPS traffic in Wireshark

The following screenshot shows a capture of DoH Traffic, and a display filter of dns and tls which narrows packets down to DoH only. It shows the resolution of A and AAAA records for isc.sans.edu

![Screenshot 2025-01-28 181128](https://github.com/user-attachments/assets/b9f5b9c6-bb39-4707-a597-71cc061ad42f)

If you use the filter (dns and tls) or http2    - this will show both DoH query and the HTTP/2 portion. You see both DoH packet and the preceding HTTP/2 packet that facilitates it. 


![Screenshot 2025-01-28 181526](https://github.com/user-attachments/assets/5bc82612-83c2-46e3-b015-d4e3981b5c22)


We notice that the DNS transaction ID is now 0x0000 for all requests. Now we need to use the HTTP/2 stream identifier. 
Lets filter traffic down to just the unique TCP connection to the DoH server and the HTTP/2 stream identifier, 
  - find packet 552
  - right-click and select follow
  - select HTTP/2 stream
  - now the following filter is applied:  tcp.stream eq 2 and http2.streamid eq 81


![Screenshot 2025-01-28 182253](https://github.com/user-attachments/assets/6e37e881-ff92-4e5a-876e-b3944e7fc131)

stream identifier can be found in the brackets after HEADERS




Now let's see what traffic looks like without the encryption keys applied. 



![Screenshot 2025-01-28 182555](https://github.com/user-attachments/assets/bcdb0e71-0518-46eb-be88-1006f6bd17ab)

Wireshark can no longer identify DoH traffic without the decryption keys applied. DNS will lok like any other TLS traffic without decryption. 


### Additional Question

Use a dig command to check the google.com CAA type record and examine the output to find the authoritative Certificate Authority that is designated by Google for which server should be issuing all certificates. 
- To do this using DNS, we check the CAA (Certificate Authority Authorization) record.
- If we issue the command "dig google.com CAA"  we receive a response that has a line that contains the word issue on it.


![Screenshot 2025-01-29 075630](https://github.com/user-attachments/assets/eff9136c-1d76-450c-8e8c-0a33078df79c)

Final line shows that Google says that ONLY pki.goog is authorized to generate a certificate for the google.com domain.
