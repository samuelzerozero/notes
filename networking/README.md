# Networking Notes

#### What the difference between a classful and classless Networks?
The classfull are the real one, and are limited to the offset on the 8 digits. A-B-C. They were used to get the public IP block space that didn't quite fit the range that was willing to buy. Classless are CIDR notation, that has the flexibility of deciding a more flexible submask.

#### How to calculate the Subnet mask, Network address,Broadcast Address and available address range from a CIDR IP?
Example IP: 27. 27. 120. 92/ 25
Subnet mask: ```25 leading 1 11111111.11111111.11111111.10000000 255.255.255.128 ```
Network Address: ``` 27. 27. 120. 9 00011011.00011011.01111000.01011100 00011011.00011011.01111000.0000000 27. 27. 120. 0```
Broadcast Address: ```00011011.00011011.01111000.01111111 255 - 128  27. 27. 120. 127```
Available Range: ```27. 27. 120. 1 - 27. 27. 120. 126```
Number of Ip available: ```32 - 25 = 7 2^7-2=126 ```
