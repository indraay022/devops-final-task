# Create Server
Buat 3 server =  
`indra-app (2 CPU, 2GB RAM)`  
untuk deploy aplikasi dan database  

`indra-monitor (2 CPU, 2GB RAM)`  
untuk monitoring server dan cicd  

`indra-nginx (1 CPU, 1GB RAM)`  
untuk web server  
![Screen Shot 2022-09-27 at 05 45 32](https://user-images.githubusercontent.com/110447286/193117724-50a8ab1b-9a8c-4a3e-afcb-f0f21d480f20.png)


# Setup SSH custom key
Masuk pada direktori `~/.ssh`  
```
cd .ssh
```
Jika belum ada direktori `.ssh` maka buat direktori baru terlebih dahulu  
```
mkdir .ssh
```
Kita buat kunci ssh dengan nama indra@local  
```
ssh-keygen
```
Lalu kita isikan nama kunci kita sebagai `indra@local`  
Kita lihat sudah terdapat 2 kunci baru yaitu `indra@local` dan `indra@local.pub`  
Selanjutnya kita sisipkan kunci `indra@local.pub` ke dalam file bernama `authorized_keys`  
```
cat indra@local.pub >> authorized_keys
```
Lalu kita cek isi dari file `authorized_keys`
```
cat authorized_keys
```
Disini bisa kita lihat kunci `indra@local.pub` sudah tertampung pada `authorized_keys`  

Selanjutnya kita buat file baru bernama `config`  
```
pico config
```
Didalam file config kita buat alias dari 3 server yang sudah kita buat  
Hal ini digunakan untuk memudahkan kita dalam melakukan remote menggunakan ssh  
Jadi kita tidak perlu memasukkan username dan ip address saat ingin berpindah remote  
Kita cukup menggunakan aliasnya saja

Selanjutnya kita kembali ke direktori home 
```
cd ~
```
Kita transfer isi dari folder `.ssh` di lokal menuju ke 3 server yang akan kita remote  
```
scp .ssh/* indra-app:~/.ssh
```
```
scp .ssh/* indra-nginx:~/.ssh
```
```
scp .ssh/* indra-monitor:~/.ssh
```
Setelah itu kita coba login ke server dengan menggunakan kunci dan alias  
```
ssh -i .ssh/indra@local indra-app
```
```
ssh -i .ssh/indra@local indra-nginx
```
```
ssh -i .ssh/indra@local indra-monitor
```
Bisa kita lihat bahwa kita dapat login ke server tanpa password dan tanpa perlu menyebutkan username dan ip addess server  

# Setup SSH (id_rsa & id_rsa.pub)
Di task kali ini saya menggunakan ssh key default yaitu `id_rsa` dan `id_rsa.pub` karena lebih mudah (tidak perlu specify custom key)  
Jadi penggunaan command ssh akan lebih simple yaitu menjadi
```
ssh <alias>
```

Pada local kita generate ssh key baru  
```
ssh-keygen
```
maka akan ada 2 file baru di direktori `.ssh` yaitu `id_rsa` (private key) dan `id_rsa.pub` (public key)  

Sama seperti pada custom key saya akan menyisipkan puclic key `id_rsa.pub` ke dalam `authorized_keys` kemudian saya mentransfer ulang `authorized_keys`, `id_rsa` dan `id_rsa.pub` ke 3 server

# Membuat User
untuk membuat user baru pada semua server kita perlu menggunakan ansible untuk meng-otomasi perintah ke dalam server  
Jadi kita dapat membuat user baru secara serentak tanpa perlu konfigurasi satu per satu  
   
Langkah awalnya kita buat folder bernama `ansible` sebagai work directory ansible kita  
Lalu di dalam direktori tersebut kita buat file baru bernama `inventory`  
File `inventory` berisi daftar host/server yang akan kita otomasi  
```
[indra_app]
103.172.204.148

[indra_monitoring]
103.174.114.139

[indra_nginx]
103.174.115.209
```
Untuk mengecek apakah host sudah dikenali oleh ansible local kita dapat melakukan ping  

```
ansible all -i inventory -m ping
```
Bisa kita lihat dari 3 server kita sudah kita ping dan hasilnya sukses
![Screen Shot 2022-09-27 at 07 25 31](https://user-images.githubusercontent.com/110447286/193118460-1a9cd9c9-fa8a-4d36-aacf-e46e1a80e9e1.png)  


Selanjunya sebelum membuat user kita generate password terlebih dahulu  
```
mkpasswd
```
Lalu kita masukkan password yang akan kita buat, kemudian kita diminta untuk masukkan lagi untuk konfirmasi  
Selanjutnya kita copy password kita yang sudah ter-enkripsi tersebut  

![Screen Shot 2022-09-27 at 07 29 28](https://user-images.githubusercontent.com/110447286/193118513-6576ef01-257c-49b0-8f6e-01acd36fdcb1.png)    
  
Kita buat file `.yml` untuk ansible-playbook bernama `create_user.yml`  
Untuk nama user kali ini kita buat `indraay`
```
- hosts: all
  become: true
  vars:
    username: indraay
  
  tasks:
        - name : creating new user
          ansible.builtin.user:
            name: '{{username}}'
            password: *****
            home: '/home/{{username}}'
```

Kita jalankan file tersebut dengan perintah
```
ansible-playbook -i inventory create_user.yml
```
![Screen Shot 2022-09-27 at 07 32 15](https://user-images.githubusercontent.com/110447286/193118645-a67d82e5-8a49-4bd0-b30b-17d483a627b9.png)  

Selanjutnya kita cek ke dalam server apakah user sudah berhasil dibuat  
```
su indraay
```
masukkan password
```
whoami
```
Bisa kita lihat disini user `indraay` sudah terdaftar pada server  
![Screen Shot 2022-09-27 at 07 33 10](https://user-images.githubusercontent.com/110447286/193118789-4ca5c1b4-b1f6-4892-9427-467f8a8a3f5e.png)  

# Setup Git lokal
Pertama kita perlu inisiasi config global username dan email terlebih dahulu  
```
git config --global user.name "indraay022"
```
```
git config --global user.email "email"
```
Untuk melihat daftar config yang sudah dibuat gunakan perintah  
```
git config --list
```
Maka akan muncul daftar konfigurasi yang di inisiasi  
![Screen Shot 2022-09-27 at 07 37 33](https://user-images.githubusercontent.com/110447286/193118843-f9d4a122-8db5-4abb-9d2e-396a930868bb.png)  


Selanjutnya kita setup koneksi antara git lokal dengan github menggunakan ssh  

Kita tampilkan isi dari `.ssh/id_rsa.pub` lalu kita copy ke dalam clipboard  
```
cat .ssh/id_rsa.pub
```
esesha

Masuk ke https://github.com/settings/keys  
Klik `SSH and GPG keys`  
Klik `New SSH key`  
![Screen Shot 2022-09-27 at 08 53 08](https://user-images.githubusercontent.com/110447286/193119164-3341aea6-c0db-4667-a925-354f35c8a221.png)  

Title  
`indra@local` (bebas)  

Key Type  
Authentication Key  

Key  
Pastekan isi dari clipboard  

Jika selesai, Klik `Add SSH Key`  
esesha
Selanjutnya kita akan diminta untuk memasukkan password sebagai konfirmasi  
![Screen Shot 2022-09-27 at 08 53 52](https://user-images.githubusercontent.com/110447286/193119202-d92a22e1-3b94-41df-8808-474227df4bcd.png)  

Bisa kita lihat disini SSH key kita sudah terdaftar  

Pada server kita tes koneksi SSH dengan perintah
```
ssh -T git@github.com
```
Jika berhasil maka outputnya adalah seperti berikut  
![Screen Shot 2022-09-27 at 08 55 55](https://user-images.githubusercontent.com/110447286/193119256-76581f98-b2f2-4e6e-9977-6e198c03fed7.png)  

# Clone Git Repository
Pada browser masuk ke repository `dumbwaysdev/literature-frontend` dan `dumbwaysdev/literature-backend`  
Lalu klik fork untuk meng-copy repositori tersebut ke repository akun GitHub kita  
![Screen Shot 2022-09-27 at 09 02 23](https://user-images.githubusercontent.com/110447286/193119383-79be8452-bb6c-40d4-be68-fd8e2fc6ac32.png)  

Setelah itu kita dapat mengedit Repository name sesuai custom kita, namun disini saya menggunakan nama yang tetap sesuai repository aslinya  
Description bersifat opsional jadi bebas  
Klik checkbox Copy the `main` branch only  
Setelah itu klik `Create Fork`
![Screen Shot 2022-09-27 at 09 02 41](https://user-images.githubusercontent.com/110447286/193119434-2400a9ed-8d30-46ec-8ed5-f0cd414055af.png)  

Lakukan hal yang sama kepada repository yang satunya  

Untuk meng-clone repository ke dalam server kita maka  
Klik `Code`  
Klik `SSH`  
Klik tombol Copy
![Screen Shot 2022-09-27 at 09 02 58](https://user-images.githubusercontent.com/110447286/193119505-e043e814-81b8-4fcc-999e-4131b8e57d82.png)  

Pada server kita masukkan perintah 
```
git clone <paste remote url>
```
![Screen Shot 2022-09-27 at 09 04 59](https://user-images.githubusercontent.com/110447286/193119549-44d500c6-b198-445d-88b9-74c9e2bf49f5.png)  

Lakukan hal yang sama kepada repository yang satunya

Maka kedua repository berhasil ter-clone ke dalam server kita  

# Make branch
Disini kita perlu membuat 3 branch baru yaitu `Development`, `Staging`, dan `Production`
  
Untuk membuat branch pada repository kita perlu masuk ke dalam folder repository tersebut  
```
cd literature-frontend
```

Lalu buat branch baru dengan perintah 
```
git branch Development
```
```
git branch Staging
```
```
git branch Production
```

Untuk melihat branch apa saja yang ada di dalam repository gunakan perintah
```
git branch -a
```

Maka bisa kita lihat kita telah membuat 3 branch baru `Development`, `Staging`, dan `Production`  
![Screen Shot 2022-09-27 at 09 09 26](https://user-images.githubusercontent.com/110447286/193119802-e658c1c9-5252-4660-a730-ebd1190a1be0.png)  

Lakukan hal yang sama ke repository yang satunya  

Kita juga dapat mengecek remote yang ada dengan perintah
```
git remote -v
```
![Screen Shot 2022-09-27 at 09 09 52](https://user-images.githubusercontent.com/110447286/193119841-ddda1b39-2ebc-4f57-922e-c142b86e3f85.png)  

Remote diperlukan untuk melakukan push atau pull antara lokal dengan Github

Untuk berpindah Branch gunakan perintah
```
git checkout <Nama Branch>
```

Lalu untuk meng-staging semua file dalam repository gunakan perintah
```
git add .
```
Untuk meng-commit atau meng-apply perubahan pada suatu file gunakan perintah
```
git commit -m "<Pesan commit>"
```

Selanjutnya kita push repository lokal kita ke branch baru yaitu `Production` di GitHub  

Maka bisa kita lihat kita sudah membuat branch baru pada GitHub
![Screen Shot 2022-09-27 at 11 01 15](https://user-images.githubusercontent.com/110447286/193119964-5c2b6ea2-8d3e-46cd-a88e-37ed92ac1b8c.png)  

Lakukan hal yang sama ke repository yang satunya

# Instalasi Docker ke 3 server
Kali ini kita menggunakan ansible untuk otomatisasi intalasi docker ke 3 server  

Buat file `install_docker.yml` untuk instalasi docker  

```
- hosts: all
  become: true
  vars: 
    username: indra

  tasks:
        - name: sudo apt update
          ansible.builtin.apt: 
            update_cache: yes

        - name: install ca-certificates, curl, gnupg, lsb-release
          ansible.builtin.apt: 
            name:
            - ca-certificates
            - curl
            - gnupg
            - lsb-release

        - name: Add Docker GPG apt Key
          apt_key:
            url: https://download.docker.com/linux/ubuntu/gpg
            state: present

        - name: Add Docker Repository
          apt_repository:
            repo: deb https://download.docker.com/linux/ubuntu focal stable
            state: present

        - name: sudo apt update
          ansible.builtin.apt: 
            update_cache: yes
        
        - name: install docker engine
          ansible.builtin.apt: 
            name:
            - docker-ce
            - docker-ce-cli 
            - containerd.io 
            - docker-compose-plugin

        - name: docker command without sudo
          ansible.builtin.shell: "sudo usermod -aG docker {{username}}"
          args:
            executable: /bin/bash
        
        - name: install python3-pip
          ansible.builtin.apt: 
            name: python3-pip
            state: latest
    
        - name: Install docker-py
          ansible.builtin.pip:
            name: docker-py
```

Jalankan dengan perintah  
```
ansible-playbook -i inventory install_docker.yml
```
![Screen Shot 2022-09-27 at 09 45 32](https://user-images.githubusercontent.com/110447286/193120052-f78894b6-af8e-4f6b-84eb-9fe4ef24319c.png)  


Setelah selesai kita coba masuk ke salah satu server  
Lalu kita cek versi docker untuk melihat apakah docker sudah terinstall  
```
docker -v
```
```
docker compose version
```
![Screen Shot 2022-09-27 at 09 52 00](https://user-images.githubusercontent.com/110447286/193120110-2a2acd49-1bfd-4247-8883-c22850d3e2e4.png)  

Lalu kita login ke HubDocker dengan perintah
```
docker login
```
Kita masukkan username dan password  
![Screen Shot 2022-09-27 at 11 15 51](https://user-images.githubusercontent.com/110447286/193120194-016b13e3-ab7f-44e4-abbf-c1d97d8a2b58.png)


# Instalasi PostgreSQL
Buat file baru `install_postgreSQL.yml`  
```
- hosts: indra_app
  become: true

  tasks:
  - name: Creating postgresSQL
    docker_container:
      name: postgres
      image: postgres
      volumes:
        - /home/indra/postgres:/var/lib/postgresql/data
      env:
        POSTGRES_DB=literature
        POSTGRES_USER=indra
        POSTGRES_PASSWORD=l0calhost
      published_ports:
        - '5432:5432'
```
Jalankan dengan perintah  
```
ansible-playbook -i inventory install_postgreSQL.yml
```
![Screen Shot 2022-09-27 at 10 23 39](https://user-images.githubusercontent.com/110447286/193120270-258aa267-40b5-4e90-a1da-b401ebe831db.png)  

Masuk ke `indra-app` lalu cek docker container yang berjalan dengan perintah
```
docker ps
```
![Screen Shot 2022-09-27 at 10 24 31](https://user-images.githubusercontent.com/110447286/193120324-97b66abd-969d-4340-aac4-f33553a72b66.png)  

Bisa kita lihat bahwa postgreSQL sudah kita deploy ke dalam docker container

# Deploy Backend App
Sebelum mulai, kita perlu install Node Version Manager terlebih dahulu  
```
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.1/install.sh | bash
```
![Screen Shot 2022-09-27 at 10 49 19](https://user-images.githubusercontent.com/110447286/193120396-7117e549-c14e-42d8-a376-70998f2e14cd.png)


Lalu kita exec bash
```
exec bash
```
Selanjutnya kita install node versi 14
```
nvm install 14
```
![Screen Shot 2022-09-27 at 10 50 24](https://user-images.githubusercontent.com/110447286/193120449-92c3cc47-a180-4e61-a3f7-454693975858.png)  

Masuk ke direktori `literature-backend`  
```
cd literature-backend
```

edit file `config/config.json`
```
nano config/config.json
```

Isi username, password, database, host, dan dialect sesuai konfigurasi database kita
![Screen Shot 2022-09-27 at 11 04 37](https://user-images.githubusercontent.com/110447286/193120546-7bf5aa01-a31c-4f30-abef-6518724021e9.png)


Sebelum build image kita lihat terlebih dahulu isi dari `package.json`  
```
cat package.json
```  
Bisa kita lihat disini untuk menjalankan app kita perlu menggunakan perintah `nodemon server.js`  
![Screen Shot 2022-09-30 at 02 07 33](https://user-images.githubusercontent.com/110447286/193120879-81e460f1-01a0-4ec6-a409-be288a01eb43.png)  

Lalu kita buat file baru bernama `Dockerfile`
```
FROM node:14
WORKDIR /home/root/
COPY . .
RUN npm install
RUN npm install -g nodemon
RUN npm install sequelize-cli
RUN npx sequelize db:migrate
EXPOSE 5000
CMD [ "nodemon", "server.js" ]
```

Setelah itu kita build image dengan perintah
```
docker build -t indraay022/literature-backend:latest .
```

Setelah image dibuat, kita dapat menjalankan container menggunakan image tesebut  
Kita perlu membuat file `docker-compose.yml` terlebih dahulu untuk menjalakan container  
```
nano docker-compose.yml
```
```
version: '3.7'
services:
 backend-literature:
   container_name: backend-literature
   image: indraay022/literature-backend:latest
   ports:
     - 5000:5000
```
Setelah itu kita jalankan perintah berikut untuk membuat container baru
```
docker compose up -d
```
Bisa kita cek melalui browser, kita buka ip kita di port 5000  
![Screen Shot 2022-09-27 at 11 34 55](https://user-images.githubusercontent.com/110447286/193121279-33a7683e-d0b3-4bdb-bc73-931a185e5dda.png)  


# Deploy Frontend App
Masuk ke direktori `literature-frontend`
```
cd literature-frontend
```
Edit file pada `src/config/config.js`
```
nano `src/config/config.js`
```
Pada baseURL kita isikan url backend app yang kita deploy  
![Screen Shot 2022-09-27 at 11 43 46](https://user-images.githubusercontent.com/110447286/193121405-4a8c483d-0f8d-4096-a372-fd07c4bd505e.png)  

Selanjutnya kita buat `Dockerfile`
```
FROM node:14-alpine
WORKDIR /home/root
COPY . .
RUN npm install
EXPOSE 3000
CMD [ "npm", "start" ]
```
Setelah itu kita build image dengan perintah
```
docker build -t indraay022/literature-frontend:latest .
```
Buat file `docker-compose.yml`  
```
version: '3.7'
services:
 frontend-literature:
   container_name: frontend-literature
   image: indraay022/literature-frontend:latest
   stdin_open: true
   ports:
     - 3000:3000
```
Setelah itu kita jalankan perintah berikut untuk membuat container baru
```
docker compose up -d
```
Bisa kita cek melalui browser, kita buka ip kita di port 3000  
![Screen Shot 2022-09-27 at 11 56 44](https://user-images.githubusercontent.com/110447286/193121513-e5f3a5b7-6dda-433a-9de7-cf64d2f3a1be.png)  


# Remote Database
Pada server lain (misal server `indra-nginx`) kita install Postgresql
```
sudo apt install postgresql
```
![Screen Shot 2022-09-27 at 12 13 13](https://user-images.githubusercontent.com/110447286/193121581-e1cb2b47-72b8-407d-941d-ea1713fa5e11.png)  

Lalu kita bisa me-remote database pada indra-app dengan perintah berikut
```
psql -h <ip indra-app> -p 5432 -U indra -d literature
```
Masukkan password  
Lalu kita sudah masuk ke database literature  
![Screen Shot 2022-09-27 at 12 13 45](https://user-images.githubusercontent.com/110447286/193121657-33014167-32b3-4d5e-b17f-4b21c007e466.png)  

# Setup WebServer, Auth, dan SSL
Pada server `indra-nginx` install aplikasi webserver `nginx`  
```
sudo apt install nginx
```
![Screen Shot 2022-09-27 at 14 12 46](https://user-images.githubusercontent.com/110447286/193121688-58f0632e-3994-472b-8c64-3dbb04466ecb.png)  

Install juga `apache2-utils`
```
sudo apt install apache2-utils
```
![Screen Shot 2022-09-28 at 20 23 41](https://user-images.githubusercontent.com/110447286/193121866-7f305df9-0a36-4897-a9ba-e5b18a73d53c.png)  

Lalu gunakan perintah berikut untuk menambahkan kredensial baru `indra`
```
sudo htpasswd -c /etc/apache2/.htpasswd indra
```
Masukkan password
![Screen Shot 2022-09-28 at 20 24 39](https://user-images.githubusercontent.com/110447286/193121907-08892be6-8852-45eb-b325-97b64500df7a.png)  

Kita lihat apakah kredensial sudah berhasil ditambahkan  
```
cat /etc/apache2/.htpasswd
```
![Screen Shot 2022-09-28 at 20 25 06](https://user-images.githubusercontent.com/110447286/193121947-f6b7ad5e-28a9-4983-910e-6db3fe6b61d4.png)  

Setelah itu kita lanjut konfigurasi SSL  
Install aplikasi `snapd` agar dapat menginstall `certbot` 
```
sudo apt install snapd
```
![Screen Shot 2022-09-27 at 14 14 06](https://user-images.githubusercontent.com/110447286/193122084-4c64b6ab-3b9a-45ec-ab4e-73377c9e2b8a.png)  

Lalu kita gunakan `snapd` untuk menginstall `certbot`
```
sudo snap install --classic certbot
```

Lalu gunakan perintah berikut untuk mengecek apakah `certbot sudah terinstal dengan benar
```
sudo ln -s /snap/bin/certbot /usr/bin/cerbot
```
![Screen Shot 2022-09-27 at 14 16 38](https://user-images.githubusercontent.com/110447286/193122109-c659b504-3274-4ece-9b78-a30f67c5b8a5.png)  

Gunakan perintah berikut untuk setup `certbot` agar dapat menjalankan plugin dengan root  
```
sudo snap set certbot trust-plugin-with-root=ok
```
![Screen Shot 2022-09-27 at 14 17 22](https://user-images.githubusercontent.com/110447286/193122321-9cdba924-e6e3-4751-975e-3db69cc562c4.png)

Lalu install plugin `certbot-dns-cloudflare`
```
sudo snap install certbot-dns-cloudflare
```
![Screen Shot 2022-09-27 at 14 18 28](https://user-images.githubusercontent.com/110447286/193122263-fd3ba0cc-b514-4910-985c-66699d363a40.png)  

Setelah itu di Cloudflare kita buat beberapa record dengan proxy status DNS Only  
Seperti berikut  
![Screen Shot 2022-09-27 at 14 31 07](https://user-images.githubusercontent.com/110447286/193122441-c43ec63d-2065-4ef9-8732-dcc9bfca0d7f.png)  

Pada profile kita buat API token untuk server kita  
Klik `Create Token`
![Screen Shot 2022-09-27 at 14 31 45](https://user-images.githubusercontent.com/110447286/193122477-af11a189-e767-436d-a780-510dcfbd82ae.png)  

Pada `Edit zone DNS` klik `Use Template`  
![Screen Shot 2022-09-27 at 14 31 55](https://user-images.githubusercontent.com/110447286/193122505-2e463aa3-2952-4ac4-9e50-5f55b47b9938.png)  

Atur seperti berikut lalu klik `Continue to summary`  
![Screen Shot 2022-09-27 at 14 34 25](https://user-images.githubusercontent.com/110447286/193122551-fbfe6a08-380f-4f98-84ad-946bd4727236.png)  
![Screen Shot 2022-09-27 at 14 34 33](https://user-images.githubusercontent.com/110447286/193122569-496ecb57-e705-4b14-a245-42aaf422e5f3.png)  

Klik `Create Tokens`
![Screen Shot 2022-09-27 at 14 34 53](https://user-images.githubusercontent.com/110447286/193122588-f08ba074-9999-4708-9936-47c791e24af0.png)  

Jalankan perintah pada field hitam di server `indra-nginx`  
Jika muncul "This API Token is valid and active" berarti sudah berhasil  
![Screen Shot 2022-09-27 at 14 35 01](https://user-images.githubusercontent.com/110447286/193122630-3a073221-20ce-4578-ac2a-5c69e941f868.png)  
![Screen Shot 2022-09-27 at 14 36 49](https://user-images.githubusercontent.com/110447286/193122687-6872f222-22b7-457e-967d-8df68f6f89de.png)  

Selanjutnya kita buat direktori baru `/root/.secrets`
```
sudo mkdir /root/.secrets
```
Kita ubah mode direktori tersebut menjadi 700
```
sudo chmod 700 /root/.secrets
```
Kemudian kita buat file baru `cloudflare.ini` dalam direktori tersebut
```
sudo nano /root/.secret/cloudflare.ini
```
![Screen Shot 2022-09-27 at 14 39 31](https://user-images.githubusercontent.com/110447286/193122748-3b9c6177-edd0-44bd-b285-65866b9cc65d.png)  

```
dns_cloudflare_api_token= <paste token dari Cloudflare>
```
![Screen Shot 2022-09-27 at 14 39 57](https://user-images.githubusercontent.com/110447286/193122780-2a2a7ad5-a3fe-4ccf-b5e1-105cda3dafff.png)  

Kemudian kita ubah mode file `cloudflare.ini` menjadi 600
```
sudo chmod 600 /root/.secrets/cloudflare.ini
```
![Screen Shot 2022-09-27 at 14 40 49](https://user-images.githubusercontent.com/110447286/193122839-e54c4996-a74f-42bf-a587-77c55ad8342c.png)  

Setelah `indra-nginx` terintegrasi dengan Cloudflare maka kita perlu membuat sertifikat baru dengan perintah berkut
```
sudo certbot certonly --dns-cloudflare --dns-cloudflare-credentials /root/.secrets/cloudflare.ini -d indra.studentdumbways.my.id -d *.indra.studentdumbways.my.id
```
![Screen Shot 2022-09-27 at 14 42 22](https://user-images.githubusercontent.com/110447286/193122887-0b025aa8-cca1-4918-bb88-ca03dc304025.png)  

Maka jika berhasil akan muncul "Successfully received certificate"  
sertifikat akan tersimpan pada `/etc/letsencrypt/live/indra.studentdumbways.my.id/fullchain.pem`  
dan untuk kuncinya ada di `/etc/letsencrypt/live/indra.studentdumbways.my.id/privkey.pem`

Lalu kita buat konfigurasi reverse proxy  
`/etc/nginx/indra/indra.conf`
```
upstream indra { 
    server 103.172.204.148:3000;
}
server { 
    listen 443 ssl http2;
    server_name indra.studentdumbways.my.id; 
    
    ssl_certificate /etc/letsencrypt/live/indra.studentdumbways.my.id/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/indra.studentdumbways.my.id/privkey.pem;

    location / { 
             proxy_pass http://indra;
    }
}
```

`/etc/nginx/indra/api.conf`
```
upstream api { 
    server 103.172.204.148:5000;
}
server { 
    listen 443 ssl http2;
    server_name api.indra.studentdumbways.my.id; 
    
    ssl_certificate /etc/letsencrypt/live/indra.studentdumbways.my.id/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/indra.studentdumbways.my.id/privkey.pem;

    location / { 
             proxy_pass http://api;
    }
}
```
`/etc/nginx/indra/prometheus.conf`
```
upstream domain { 
    server 103.174.114.139:9090;
}
server { 
    listen 443 ssl http2;
    server_name prometheus.indra.studentdumbways.my.id; 
    
    ssl_certificate /etc/letsencrypt/live/indra.studentdumbways.my.id/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/indra.studentdumbways.my.id/privkey.pem;

    location / { 
             auth_basic "Not For Public!";
             auth_basic_user_file /etc/apache2/.htpasswd;
             proxy_pass http://domain;
    }
}
```
`/etc/nginx/indra/monitoring.conf`
```
upstream monitoring { 
    server 103.174.114.139:3000;
}
server { 
    listen 443 ssl http2;
    server_name monitoring.indra.studentdumbways.my.id; 
    
    ssl_certificate /etc/letsencrypt/live/indra.studentdumbways.my.id/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/indra.studentdumbways.my.id/privkey.pem;

    location / { 
             proxy_set_header Host monitoring.indra.studentdumbways.my.id;
             proxy_pass http://monitoring;
    }
}

```
`/etc/nginx/indra/cicd.conf`
```
upstream cicd { 
    server 103.174.114.139:8080;
}
server { 
    listen 443 ssl http2;
    server_name cicd.indra.studentdumbways.my.id; 
    
    ssl_certificate /etc/letsencrypt/live/indra.studentdumbways.my.id/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/indra.studentdumbways.my.id/privkey.pem;

    location / { 
             proxy_pass http://cicd;
    }
}
```

Kemudian kita include-kan semua file yang ada di direktori `/etc/nginx/indra` ke dalam sebuah file pada `/etc/nginx/nginx.conf`  
![Screen Shot 2022-09-27 at 14 56 42](https://user-images.githubusercontent.com/110447286/193123003-894aef8f-8981-42dc-a103-1977534e2eb7.png)  

Kita cek konfigurasi nginx dengan perintah  
```
sudo nginx -t
```
Lalu kita restart nginx
```
sudo systemctl restart nginx
```
kita lihat status nginx  
```
sudo systemctl status nginx
```
![Screen Shot 2022-09-27 at 15 04 58](https://user-images.githubusercontent.com/110447286/193123052-508d4b26-4e8d-4051-ab02-92b154c74ba5.png)  

Lalu kita coba buka url dari frontend app dan backend app 
Disini bisa kita lihat bahwa koneksi sudah secure   
![Screen Shot 2022-09-27 at 15 05 40](https://user-images.githubusercontent.com/110447286/193123097-c1ef99d7-2e42-4ce0-9dcc-152bbef21397.png)  
![Screen Shot 2022-09-27 at 15 16 48](https://user-images.githubusercontent.com/110447286/193123129-adcece0f-bf88-47f3-bf29-45a9e8e446a8.png)  


# Install Monitoring App
Aplikasi pertama yaitu `node exporter`
Kita gunakan ansible untuk instalasi docker containernya ke 3 server

Buat file baru `install_nodexp.yml`
```
- hosts: all
  become: true

  tasks:
  - name: Creating node exporter container
    docker_container:
      api_version: "1.39"
      name: node-exporter
      image: prom/node-exporter:v0.18.1
      volumes:
        - /proc:/host/proc:ro
        - /sys:/host/sys:ro
        - /:/rootfs:ro
        - /var/lib/docker:/var/lib/docker:ro
      command:
        - '--path.procfs=/host/proc'
        - '--path.rootfs=/rootfs'
        - '--path.sysfs=/host/sys'
        - '--collector.filesystem.ignored-mount-points=^/(sys|proc|dev|host|etc)($$|/)'
      restart_policy: unless-stopped
      published_ports:
        - "9100:9100"

```
Jalankan dengan perintah
```
ansible-playbook -i inventory install_nodexp.yml
```
![Screen Shot 2022-09-27 at 22 04 06](https://user-images.githubusercontent.com/110447286/193123238-4cab8073-5161-487c-bdc3-3dcf15e3ae69.png)  

Aplikasi selanjutnya yaitu `Prometheus` dan `Grafana`  
Sebelumnya kita perlu mengatur agar docker pada masing-masing server agar meng-export docker metrics-nya pada port 9323  

Caranya buat file baru `/etc/docker/docker.json` pada masing-masing server  
```
{
  "metrics-addr" : "<ip server>:9323",
  "experimental" : true
}
```

Selanjutnya server `indra-monitoring` kita buat direktori baru `prometheus`
```
mkdir prometheus
```
Didalam direktori tersebut buat file baru bernama `prometheus.yml`
```
nano prometheus/prometheus.yml
```
![Screen Shot 2022-09-27 at 22 15 21](https://user-images.githubusercontent.com/110447286/193123356-793e0191-76ed-4187-872a-77db134583d8.png)

`~/prometheus/prometheus.yml`
```
global:
  scrape_interval: 15s
  evaluation_interval: 5s
  scrape_timeout: 10s
  external_labels:
    monitor: 'codelab-monitor'

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['103.174.114.139:9090']

  - job_name: 'docker metrics'
    static_configs:
      - targets: ['103.172.204.148:9323','103.174.114.139:9323','103.174.115.209:9323']

  - job_name: 'node_exporter metrics'
    static_configs:
      - targets: ['103.172.204.148:9100','103.174.114.139:9100','103.174.115.209:9100']
```

Selanjutnya kita gunakan ansible untuk deploy aplikasi `prometheus` dan `grafana` dengan docker container

Buat file `install_prom_graf.yml`
```
- hosts: indra_monitoring
  become: true

  tasks:
  - name: 'Creating prometheus container'
    docker_container:
      name: prometheus
      image: prom/prometheus
      volumes:
        - /home/indra/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      restart_policy: unless-stopped
      published_ports:
        - 9090:9090
  
  - name: 'Creating grafana container'
    docker_container:
      name: grafana
      image: grafana/grafana
      restart_policy: unless-stopped
      published_ports:
        - 3000:3000
```

Lalu jalankan dengan perintah
```
ansible-playbook -i iventory install_prom_graf.yml
```
![Screen Shot 2022-09-27 at 22 38 47](https://user-images.githubusercontent.com/110447286/193123512-7c74e159-6f9c-4c94-92a2-22d62ab11afb.png)  

Kita coba buka url aplikasi prometheus dan grafana melaui browser
![Screen Shot 2022-09-30 at 02 22 50](https://user-images.githubusercontent.com/110447286/193123744-95e8c399-6fd7-469c-b764-86ba88a23b41.png)  
![Screen Shot 2022-09-30 at 02 24 04](https://user-images.githubusercontent.com/110447286/193124024-6326e082-cb0f-4f57-9978-72853d341531.png)  
![Screen Shot 2022-09-28 at 17 27 30](https://user-images.githubusercontent.com/110447286/193123970-e994305f-8378-4d82-9b53-cb1a7874b43e.png)  



# Instalasi Jenkins
Untuk setup awal kita gunakan perintah berikut
```
docker run --name jenkins -u 0 -p 8080:8080 -v /home/indra/jenkins:/var/jenkins_home -d jenkins/jenkins:latest
```
![Screen Shot 2022-09-28 at 19 12 27](https://user-images.githubusercontent.com/110447286/193124340-453b4659-4afe-4dec-a8f3-b134a0ab5d44.png)  

Gunakan perintah berikut untuk melihat logs instalasi container jenkins
```
docker logs jenkins
```
Disini terdapat password yang perlu kita copy
![Screen Shot 2022-09-28 at 19 13 10](https://user-images.githubusercontent.com/110447286/193124382-15a1e3a9-f42a-4599-9f12-c2f49918307b.png)  

Selanjutnya kita buka url jenkins pada browser  
Kita paste password di kolom administrator password   
Klik `Continue`  
![Screen Shot 2022-09-28 at 19 14 08](https://user-images.githubusercontent.com/110447286/193124426-5e212108-f927-450b-ac78-7a4d0c63b9c9.png)  

Klik `Install suggested plugins`
![Screen Shot 2022-09-28 at 19 14 22](https://user-images.githubusercontent.com/110447286/193124453-a2418b28-3d7d-4fb5-ab36-06ae6c385af5.png)  

Tunggu hingga proses selesai  
![Screen Shot 2022-09-28 at 19 18 09](https://user-images.githubusercontent.com/110447286/193124517-7e1de51b-9dc7-481c-b4f6-4276877fe733.png)  


Create First Admin User  
Username  
`indra`

Password  
`password`

Full name  
`Indra AY`

Email  
`email`

Klik `Save and Continue`  
![Screen Shot 2022-09-28 at 19 28 42](https://user-images.githubusercontent.com/110447286/193124554-33a4fa84-e0a8-47f5-ad5c-189f3726e486.png)  

Jenkins URL isikan sesuai url saat ini  
Klik `Save and Finish`  
![Screen Shot 2022-09-28 at 19 29 08](https://user-images.githubusercontent.com/110447286/193124574-5399b24b-04eb-42d8-8817-ba34df105db4.png)  

Klik `Start using Jenkins`  
![Screen Shot 2022-09-28 at 19 29 14](https://user-images.githubusercontent.com/110447286/193124602-d826dee2-36cb-451d-ae20-6877fd101d94.png)  
