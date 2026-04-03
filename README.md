Distributed Synchronization & Deadlock Lab

Level 1: Provision Virtual Storage


The output shows /dev/loop are actively mounted loop devices backed by image files, confirming the virtual drives are successfully attached to the filesystem



<img width="1214" height="159" alt="image" src="https://github.com/user-attachments/assets/ea10cf97-7f70-4ef4-a8d0-dc86749b81e3" />

---

Level 2: The Naive Sync

Two scripts sync_up and sync_down were written to simulate bi-directional vault synchronization. sync_up locks Alpha first then Beta, while sync_down locks Beta first then Alpha. This opposite lock ordering is the critical flaw when both scripts run concurrently, they create the conditions necessary for a deadlock.


Level 3: Triggering the Deadlock

Both scripts were executed simultaneously in two separate terminals. sync_up successfully acquired the Alpha lock (fd 200) and waited for Beta. At the same time, sync_down successfully acquired the Beta lock (fd 201) and waited for Alpha. Neither process will release what it holds until it gets what it needs resulting in infinite mutual blocking. This is a classic Circular Wait deadlock satisfying all four Coffman conditions:

Mutual Exclusion — only one process can hold each lock
Hold and Wait — each process holds one lock while waiting for the other
No Preemption — neither process is forced to release its lock
Circular Wait — Process A waits for B's resource, Process B waits for A's resource

<img width="685" height="145" alt="image" src="https://github.com/user-attachments/assets/c01b6c7e-b96c-4a30-873d-91a2551cf534" />



<img width="701" height="179" alt="image" src="https://github.com/user-attachments/assets/5e272480-03c6-4b24-b70c-523b88a0ff26" />

---



<img width="1144" height="127" alt="image" src="https://github.com/user-attachments/assets/f2d40e0a-bde0-489a-a53c-3ae92322e3a1" />


---


Level 4: Multiplayer Cross-User Deadlock

g11-chan-sokseyha as Player A and g11-kong-monineath as Player B) each locked their own vault and reached across the server to lock the other's vault simultaneously. Both processes froze, simulating a distributed deadlock. This resembles a Distributed Denial of Service because two independent systems mutually block each other, making both user environments completely unresponsive.


<img width="1864" height="174" alt="image" src="https://github.com/user-attachments/assets/820c3429-e447-41c4-b59c-1417d900c966" />


---




Level 5: Global Resource Ordering Patch


<img width="1280" height="156" alt="image" src="https://github.com/user-attachments/assets/04cbb08c-6154-4b40-b848-eaf087510b23" />

A global rule was enforced: Alpha lock must always be acquired before Beta, regardless of sync direction. Player B's script was rewritten to request Alpha first even though it syncs from Beta. This breaks the Circular Wait condition because both processes now request resources in the same order — Player B must wait for Alpha before touching Beta, so only one process holds resources at a time and deadlock becomes mathematically impossible.







Level 6: Deadlock Recovery with Timeout

A new script sync_timeout was written using flock -w 5, which waits a maximum of 5 seconds for the Alpha lock. When sync_up was already holding the lock, sync_timeout waited 5 seconds, failed to acquire it, and cleanly printed an error message instead of freezing. This breaks the No Preemption deadlock condition — the process self-aborts, releases system resources, and keeps the server healthy and responsive.


<img width="811" height="259" alt="image" src="https://github.com/user-attachments/assets/8f0a849b-5714-40ce-88cf-4d069ab00ef6" />


---





Level 7: Safe Ejection (Teardown)

<img width="959" height="217" alt="image" src="https://github.com/user-attachments/assets/f8a9522f-460e-4be5-bbca-6a436f338527" />

The teardown script safely unmounts both virtual vaults using udisksctl unmount, detaches the loop devices from the kernel using udisksctl loop-delete, and removes the symlinks. The df -h | grep loop output confirms no loop devices remain mounted. Proper teardown is critical because orphaned loop devices consume kernel resources and on a shared server can exhaust the finite number of available loop device slots, preventing other users from mounting new devices.


