
It defines desirable property of a distributed system with replicated data.

C -> consistency
A -> Availability
P -> Partition tolerance


It is not possible to get all 3 features in a system.

either we get  C + A or A + P or P + C

### Consistency:
if an application writes in node A and after that queries in node B (or node A) it should get back the newly written data. 

### Availability:
All nodes of a distributed system must be available all the time.

### Partition tolerance:
If the communication breaks between distributed systems then also the application will be able to perform actions (read write etc) in the distributed system. 


>In Modern distributed systems we trade off between consistency and availability. Partition tolerance is always required. 






