---
title: "Github Enterprise Server First Setup"
description:
date: 2023-10-27T09:19:02+08:00
image:
math:
license:
hidden: false
comments: true
draft: false
categories:
  - GitHub
tags:
  - github
  - ghes
---

## Introduction

We are trying GitHub Enterprise, an enterprise version of GitHub.

GitHub Enterprise exists 2 versions: GitHub Enterprise Cloud (GHEC) and GitHub Enterprise Server (GHES). If you are a small or medium-sized organization, you can use GHEC, which is a hosted version of GitHub. If your source code is sensitive, you can use GHES, which is a self-hosted version of GitHub.

In this post, we will try GHES, which is a self-hosted version of GitHub.

## Setup VM

### Hardware Requirements

Below is the hardware requirements for GHES. (taken from [official documentation](https://docs.github.com/en/enterprise-server@3.10/admin/installation/setting-up-a-github-enterprise-server-instance/installing-github-enterprise-server-on-hyper-v))

|User licenses|x86-64 vCPUs|Memory|Root storage|Attached (data) storage|
|:--|--:|--:|--:|--:|
|Trial, demo, or 10 light users|4|32 GB|200 GB|150 GB|
|10 to 3,000|8|48 GB|200 GB|300 GB|
|3,000 to 5000|12|64 GB|200 GB|500 GB|
|5,000 to 8000|16|96 GB|200 GB|750 GB|
|8,000 to 10,000+|20|160 GB|200 GB|1000 GB|

If you need to enable GitHub Actions, you will need to add more CPU and memory.

### Setup VM

GitHub Enterpsie Server is shipped as a Virtual Appliance (OVA) file. You can download it from the [official website](https://enterprise.github.com/download).

Attach the downloaded OVA file to a virtual machine software (e.g. VMware Workstation, VirtualBox).

Start the VM!

> Note: the setup address shown in the screenshot is misleading, more on that later.

![](2024-04-21-09-24-19.png)

Swtich to other tty, you can see the word "Debian".

![](2024-04-21-09-25-08.png)

### Setup GHES

Let's connect to the GHES instance by accessing it via a web browser. The url in my case is `https://169.254.222.200/setup` (notice it's "https", not "http")

The first thing do to is upload a license file.

![](2024-04-21-09-27-00.png)

we can get license file from GHEC Settings (you can contact with GitHub's sales team for trial license)

![](2024-04-21-09-28-49.png)

upload it, set a password 'I chose `Thread-Lapped-Sleet6`', we made to next step!

Select "New Install"

![](2024-04-21-09-29-20.png)

no hostname, well, probably an issue
no other features required at the moment, select "Save and Continue"

![](2024-04-21-09-29-29.png)


Then a long list of settings, I skipped them for the moment (except SSH key)

![](2024-04-21-09-29-51.png)

Starting setup...

![](2024-04-21-09-30-17.png)

wait for about 15 minutes, all done. Click "Visit you instance"

![](2024-04-21-09-30-26.png)

first set a admin account, check "Help me set up an organization next"

![](2024-04-21-09-30-38.png)

Next is to set up an organization

![](2024-04-21-09-31-42.png)

OK GHES setup is done!.

---

Below are some screenshots of admin section:

- File storage:

  ![](2024-04-21-09-32-12.png)

- Audit log data retention period can be set for 3 months, 6 months, 9 months, 12 months, forever. I chose 6 months just like what GHEC does.

  ![](2024-04-21-09-32-41.png)

- SSH key can be set for individual user
  ![](2024-04-21-09-34-05.png)

- No IP allow list feature in GHES
  ![](2024-04-21-09-34-32.png)

    - This feature is available in GHEC
      ![](2024-04-21-09-34-46.png)

---

Let's login to the GHES instance with SSH:
```
ssh admin@169.254.222.200 -i .\id_ghes_test -p 122
```

Basic info:
```
admin@169-254-222-200:~$ uname -a
Linux 169-254-222-200 5.10.0-0.deb10.24-amd64 #1 SMP Debian 5.10.179-5~deb10u1 (2023-08-08) x86_64 GNU/Linux
admin@169-254-222-200:~$ lsb_release -a
No LSB modules are available.
Distributor ID: Debian
Description:    Debian GNU/Linux 10 (buster)
Release:        10
Codename:       buster
```

It eat up 14.2GB of memory, I haven't set any other feature yet.
![](2024-04-21-09-36-27.png)

The instance has LOTS OF containers:

```
admin@169-254-222-200:~$ docker ps
CONTAINER ID   IMAGE                                                              COMMAND                  CREATED             STATUS                       PORTS     NAMES
b67a1e03d977   viewscreen:05a21da285e02e23b1e712e8c5a2cddc3620edf6                "/bin/viewscreen"        54 minutes ago      Up 54 minutes                          viewscreen-d5b11a11-bbd7-90a5-1dba-684e2b308a70
342a8c4817ef   treelights:24ba238ab803276dab642b4088e1d7719c22aea3                "/data/bin/treelights"   54 minutes ago      Up 54 minutes                          treelights-306fcb53-e836-4a5a-e53c-03ffdcc08f79
131368a193c8   svnbridge:9bfb9c060afad68004d212bd367144e5b915df52                 "sh -c 'cd /data/svn…"   54 minutes ago      Up 54 minutes                          svnbridge-65c316a7-f333-5950-f8e4-f999eece6f1e
38809b680f16   spokesd:20a490127e03f2272056a766a88f4b681ee0ce0a                   "/spokes-sweeper"        54 minutes ago      Up 54 minutes                          spokes-sweeper-0f137886-f273-7b61-eb3a-d82b00d5789b
7adf70158d58   spokesd:20a490127e03f2272056a766a88f4b681ee0ce0a                   "/spokesd"               54 minutes ago      Up 54 minutes                          spokesd-2c998e02-1453-9dd4-f3ee-3130076e0983
0441e1032907   pages:5e250d8802876d8c0b71a7d4ebafdbe69ae556c0                     "/build/build-server…"   54 minutes ago      Up 54 minutes                          pages-b1b83ad4-85ae-a684-c11b-463f0a5176c0
5137138edc9c   notebooks:187dbe525005fa341e35866c22250669bca8a473                 "script/server"          54 minutes ago      Up 54 minutes                          notebooks-2272e596-a349-70b5-c36f-ea2fe2d6d5f8
a4e060311e65   mail-replies:c6df854c046c41835acab1ada2edfc295edebae1              "/bin/sh -c './bin/r…"   54 minutes ago      Up 54 minutes                          mail-replies-568d5ff8-5e9e-f91c-c90e-1f4e63871ee2
23a66ddea82d   lfs-server:e6ef2cbc7b0d6a49382601014db8bb3458b4ebce                "/bin/bash -c /app/l…"   54 minutes ago      Up 54 minutes                          lfs-server-ecb24a9f-78fc-c395-5e66-6c898a7a971a
82088bfc3945   kredz:eaade3e1ada18291e41f0caa043523fbc34a4805                     "varz"                   54 minutes ago      Up 54 minutes                          kredz-varz-7c11de2d-e76c-9a84-4b27-ca3fa088cad1
22d5c9538985   kredz:eaade3e1ada18291e41f0caa043523fbc34a4805                     "hydro-consumer"         54 minutes ago      Up 54 minutes                          kredz-hydro-consumer-ef41e96a-94c1-bcec-fc1a-8ea3cd3f803a
4e273a053b21   kredz:eaade3e1ada18291e41f0caa043523fbc34a4805                     "credz"                  54 minutes ago      Up 54 minutes                          kredz-a00215cb-6ce6-f30c-a90f-5e2991f3963d
e66ea8cbd55d   hookshot-go:3468dd8c194db51eb0c2797daded63fda07985bd               "/hookshot-go"           55 minutes ago      Up 55 minutes                          hookshot-go-ad8de242-a6f7-45bf-022b-42930ff5ed8e
f5319d5f1666   ghe-grafana:0de57a4011bd66140b40d627eecddec27a1fe95f               "/run.sh"                55 minutes ago      Up 55 minutes                          grafana-5df88d53-7209-f3a7-e9cc-6b5e9187f3a7
d54cb91758ff   gpgverify:6b8aa8a56ac1905865c523d239c44fef93d4bf4b                 "/build/bin/gpgverif…"   55 minutes ago      Up 55 minutes                          gpgverify-8f27cff5-9131-57df-be52-859b381c0a01
9cf240fbfe40   gitrpcd:c1c1bde3dc8b80373d896146bac51228b6f651ca                   "/usr/bin/dumb-init …"   55 minutes ago      Up 55 minutes                          gitrpcd-fd18b860-3304-4ed7-6855-2963466030af
abc24bca5b90   github:d7984739735cbb6a2f8281a1519db38487840cda                    "/usr/bin/dumb-init …"   56 minutes ago      Up 56 minutes                          github-unicorn-209d5728-b169-08cb-e000-13619d8476f0
fed3bc28d291   github:d7984739735cbb6a2f8281a1519db38487840cda                    "/usr/bin/dumb-init …"   56 minutes ago      Up 56 minutes                          github-timerd-5c6d54dc-1092-b51e-ff9c-553a5b28487d
3c5014297fc2   github:d7984739735cbb6a2f8281a1519db38487840cda                    "/usr/bin/dumb-init …"   56 minutes ago      Up 56 minutes                          github-stream-processors-98d53c61-6eed-87d0-029d-028f36e1b71c
e8d7946558b2   github:d7984739735cbb6a2f8281a1519db38487840cda                    "/usr/bin/dumb-init …"   56 minutes ago      Up 56 minutes                          github-resqued-9855932c-eb2b-eff5-b28a-4a9b79ceced8
dd078ee15a13   github:d7984739735cbb6a2f8281a1519db38487840cda                    "/usr/bin/dumb-init …"   57 minutes ago      Up 57 minutes                          github-gitauth-d60677ba-b0a1-8ff2-5deb-2e486a1246df
4c61fcc10d7b   github:d7984739735cbb6a2f8281a1519db38487840cda                    "/usr/bin/dumb-init …"   57 minutes ago      Up 57 minutes                          github-ernicorn-24b11066-73bc-35d6-da55-afa721ae6e50
711d00d6bcc0   driftwood:4399a867da16140954cff65b9e8f0bec0129f352                 "/driftwood-ghes-str…"   57 minutes ago      Up 57 minutes                          driftwood-1ebd0f09-aaf7-4bbd-867c-c83581a43b0b
d2a6e34a4112   codeload:58fd33f376183d7f4069c888981e8b109459e093                  "./codeload --config…"   57 minutes ago      Up 57 minutes                          codeload-75746569-b4db-4f8e-8059-862f9c64001f
856e4aa6f57c   babeld:6732d28259671ce18e46eaa967bdb52a7c2758cc                    "/bin/bash -c /etc/b…"   57 minutes ago      Up 57 minutes                          babeld-2ca1d53e-9126-e2cf-3e43-8efcadf8d528
ea3719fab0b2   babeld2hydro:3df1dc4489214c8a2be5694f98d72a2a6057706c              "/app/babeld2hydro"      57 minutes ago      Up 57 minutes                          babeld2hydro-cb51ccf7-8dc0-9d3a-6f1d-d45f5e9e5d3c
7aa0e53fd73c   authzd:e3caea835dc7e5470280a848531eafa58d6e2e15                    "/authzd"                57 minutes ago      Up 57 minutes                          authzd-be2345eb-250e-b73e-87f3-1845b8725834
271e845c2ff7   authnd:3ce612f06820074fdbd007f0588e569405087906                    "/app/bin/authnd"        58 minutes ago      Up 58 minutes                          authnd-44d71086-cd06-8782-8c4a-b83a12a86929
b9854e770c46   alive:46e1d36744991c81d503b7116b3707de1da166f6                     "/go/bin/alive-local"    58 minutes ago      Up 58 minutes                          alive-0537c472-1dc7-35df-b8ed-9a5365309b8d
66d98fdbac8c   kafka-lite:6e8987e3d0054cfe5a747b5c722a0f2ad7ad64d5                "/kafka-lite --log-f…"   About an hour ago   Up About an hour                       kafka-lite-cc70cce5-7f41-1499-0029-c125be516985
0104da7a0ab7   aqueduct-lite:5bacd35a04565f339348a9de9ee4fe2c0b4b0918             "bin/aqueduct-lite -…"   About an hour ago   Up About an hour                       aqueduct-lite-edb25c36-6543-8fd1-5c8e-e89b4e7518bb
a616b54b73a2   haproxy:82541810fee8914b6a7c64a86ad971dc6fa037da                   "/usr/local/bin/entr…"   About an hour ago   Up About an hour                       haproxy-cluster-proxy-1e08ac14-726c-c294-28da-d568ea4e64f2
508c98d754fc   spokesd:20a490127e03f2272056a766a88f4b681ee0ce0a                   "/usr/bin/dumb-init …"   About an hour ago   Up About an hour                       spokesctl-7c547002-9110-f46e-50c6-e15991d05c02
999b218465e7   redis-container:0c18b5d5f2bc6f4dbea199ec0e89437df9330606           "docker-entrypoint.s…"   About an hour ago   Up About an hour                       redis-bce557dd-4cd5-9e72-8d23-47180ea83346
ec24e72202e9   ghe-nginx:990760e1d8f278d133826d0b34ed0816e65cc8b1                 "/usr/bin/dumb-init …"   About an hour ago   Up About an hour                       nginx-81dd14c3-dfcd-cd40-08c0-fe302a4fbffa
703221f65efc   ghe-mysql-container:591841296a64f8eac0ee54506b03025225d5f998       "/entrypoint.sh mysq…"   About an hour ago   Up About an hour                       mysql-33421cd2-b905-c691-9032-101612fc1e27
d0526a4372b3   memcached:1aaaa7957181c1cae709cc83aac6a6d647caef53                 "memcached -p 11211 …"   About an hour ago   Up About an hour (healthy)             memcached-dd931c40-3970-8e84-ba77-1b31537aeafb
d0bbe5b9e4d4   http2hydro:52759cc004e74aeb2c643023cc5bf45c922bcef7                "/bin/http2hydro"        About an hour ago   Up About an hour                       http2hydro-92cd5244-d507-0f1d-5fe8-e5cd16c81022
363e27a36478   haproxy:82541810fee8914b6a7c64a86ad971dc6fa037da                   "/usr/local/bin/entr…"   About an hour ago   Up About an hour                       haproxy-frontend-69ca04a8-4a0c-a112-0736-5c4ae08e2121
72e971a518c4   haproxy:82541810fee8914b6a7c64a86ad971dc6fa037da                   "/usr/local/bin/entr…"   About an hour ago   Up About an hour                       haproxy-data-proxy-7e483b28-2395-02d0-ef89-483bf9736125
a8e3b8523988   ghe-graphite-web:b30e4c827eff3d5fd22336842b196e2f18b32af9          "/entrypoint.sh --py…"   About an hour ago   Up About an hour                       graphite-web-36c0b024-3bde-8cb9-ad34-e622eba60d1e
62ef9d2dce60   gitmon:3efdfa2414b2d51dcb3d6ad822da7172307263c1                    "/build/bin/gitmon -…"   About an hour ago   Up About an hour                       governor-8cf53f5d-5306-6821-fa68-475cd5b51657
3fa700fd0c2c   github:d7984739735cbb6a2f8281a1519db38487840cda                    "/usr/bin/dumb-init …"   About an hour ago   Up About an hour                       git-daemon-d85f100b-ec9d-1d6c-8782-c213015e5d67
f2b278cbc6ef   github:d7984739735cbb6a2f8281a1519db38487840cda                    "/usr/bin/dumb-init …"   About an hour ago   Up About an hour                       github-env-1763e5b3-da55-0657-dc64-aeeaeb3ff822
d24c9b2f476b   elasticsearch-container:ff4e8a8a1dfcc086ab16ed159f4e9270f42bb14a   "elasticsearch -Edef…"   About an hour ago   Up About an hour                       elasticsearch-c4cab4a3-00e3-d81e-19f3-7df55407c9a7
da55ca10990e   consul-template:2ccb7ecaca5ee3f797b56777376b116c2d7250c4           "/entrypoint.sh -con…"   About an hour ago   Up About an hour                       consul-template-003180e7-3830-21bf-c68b-ea93a251aa0b
7f132cbe4122   alambic:4453ec114da1ad62425911118ae03a43bafc9087                   "/build/bin/alambic"     About an hour ago   Up About an hour                       alambic-39c6915c-7e3b-83ce-317c-288819fe6302
```

