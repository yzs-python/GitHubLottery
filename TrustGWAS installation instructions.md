## TrustGWAS installation instructions



### Prepare

Two computers (or virtual machines): CPU4 core or above, memory 8GB or above, hard disk 100GB or above

Operating system: centos7.9.2009, used in this article

http://mirrors.bupt.edu.cn/centos/7.9.2009/isos/x86_64/CentOS-7-x86_64-DVD-2009.iso

Network: Suppose the ip of the two computers is 192.168.31.210, 192.168.31.211 (set according to the actual situation), and the two computers can communicate with each other.

Installation package: TrustoneGWAS .zip

### Installation package file description

Decompression

```bash
unzip TrustoneGWAS.zip
cd TrustoneGWAS
```

```bash
└── TrustoneGWAS
    ├── config.conf    --- Configuration files
    ├── docker-deploy  --- docker image deployment script
    ├── dockers        --- docker images
    ├── install.sh     --- One-click installation script
    └── pca            --- PCA related procedures

```

### Configuration

config.conf must be configured before installation, on computers with IP **192.168.31.210** 

```bash
#!/bin/bash

party_list=(10000 9999)
party_ip_list=(192.168.31.210 192.168.31.211)
my_party=10000
```

Generally speaking, you only need to modify the 2 ip addresses in the party_ip_list to your actual ip

In addition, when installing on computers with ip as **192.168.31.211** , only need to modify my_party to 9999

```bash
#!/bin/bash

party_list=(10000 9999)
party_ip_list=(192.168.31.210 192.168.31.211)
my_party=9999
```

### Installation

Switch to root and execute install.sh script

```bash
cd TrustoneGWAS
bash install.sh
```

After installing on both computers, you still need to make some configurations. Select one of the two computers as the Server. Here, select the computer with IP **192.168.31.211** as the Server. Here is how to configure it

On a computer with **192.168.31.210** IP

```bash
docker exec -it confs-10000_python_1 bash
vi config.xml
# change ip to 192.168.31.211
```

On a computer with **192.168.31.211** IP

```bash
docker exec -it confs-9999_python_1 bash
vi config.xml
# change ip to 192.168.31.211
```

If the installation process goes well, you can skip the installation script instructions

### * Installation script instructions

```bash
# set env path
export TrustoneGWASPath=$(pwd)
export FateInstallPath=/usr/TrustoneGWAS/fate/
echo 'install dependence...'

# close firewall
sudo systemctl stop firewalld.service
sudo systemctl disable firewalld.service

# install docker
sudo yum update -y
sudo yum install -y yum-utils
sudo yum install -y device-mapper-persistent-data
sudo yum install -y lvm2
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install -y docker-ce
sudo systemctl start docker
sudo systemctl enable docker

# install docker-compose
sudo curl -L "https://github.com/docker/compose/releases/download/1.25.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose

# load docker
echo 'load docker...'
docker load -i ./dockers/he-python-v5.tar
docker load -i ./dockers/client.tar
docker load -i ./dockers/eggroll.tar
docker load -i ./dockers/fateboard.tar
docker load -i ./dockers/serving-proxy.tar
docker load -i ./dockers/serving-server.tar
docker load -i ./dockers/redis.tar
docker load -i ./dockers/mysql.tar

# generate docker config files
bash ./docker-deploy/generate_config.sh

# install config
sudo mkdir -p ${FateInstallPath}
source ./config.conf
sudo cp ./docker-deploy/outputs/*${my_party}*.tar ${FateInstallPath}
tar -xf ${FateInstallPath}confs-${my_party}.tar -C ${FateInstallPath}
tar -xf ${FateInstallPath}serving-${my_party}.tar -C ${FateInstallPath}

# start docker
cd ${FateInstallPath}confs-${my_party}
docker-compose up -d
cd ${FateInstallPath}serving-${my_party}
docker-compose up -d

# install pca
cd ${TrustoneGWASPath}/pca
bash install.sh
```

### Test

#### Enter docker container

On the **192.168.31.210**

```bash
docker exec -it confs-10000_python_1 bash
```

On the **192.168.31.211**

```bash
docker exec -it confs-9999_python_1 bash
```

#### Start trustoneServer

On a PC with **192.168.31.211 (the server selected in the installation step above)** ip

```nginx
nohup ./trustoneServer &
```

####   Execute script

In docker, two sets of data are prepared, test_data1 and test_data2.

- test_data1

  On the **192.168.31.210**

  ```bash
  bash run_lead_test_data1.sh
  ```

  On the **192.168.31.211**

  ```bash
  bash run_main_test_data1.sh
  ```

  Next, just wait patiently for the result of the algorith

- test_data2

  Similarly test_data1, the corresponding execution script suffix is test_data2

### * TrustGWAS command parameter

The scripts used in the above tests are all configured in advance. The detailed parameter configuration is as follows

**All mandatory parameters are as follows:**

| --file   | {filepath}  | Specify the path to the ped/map file                         |
| -------- | ----------- | ------------------------------------------------------------ |
| --bfile  | {filepath}  | Specify the path to the bed/fam/bim file                     |
| --afile  | {filepath}  | Specify another side map file (for  loci alignment)          |
| --abfile | {filepath}  | Specify the other side of the bim  file (for loci alignment) |
| --acount | {count}     | Specify the total number of  samples from the other side of the sample |
| --role   | {lead/main} | Specify the user role: lead  initiator/main responder        |

**All optional parameters are as follows：**

| --all      |         | Run the program with all the  following parameters and their default values |
| ---------- | ------- | ------------------------------------------------------------ |
| --mind     | {0.1}   | Calculate the maximum individual  missing rate (default: 0.1) |
| --gene     | {0.1}   | Calculate the maximum sample  missing rate (default: 0.1)    |
| --maf      | {0.05}  | Calculate MAF (default: 0.05)                                |
| --hwe      | {0.005} | Calculate the HWE (default: 0.005)                           |
| --ld       | {0.2}   | Calculate LD (default: r2 = 0.2)                             |
| --ld-win   | {1000}  | Set LD window size (default:  1000kb)                        |
| - it       | {0.5}   | Calculate heterozygosity (default: 0.5)                      |
| --chisk    | {0.05}  | Calculated chi-square (default: p <= 0.05)                   |
| --catt     | {0.05}  | Calculation of CATT (default: p <= 0.05)                     |
| --pca      | {5}     | Calculate PCA principal components (default: 5)              |
| --linear   | {0.05}  | Calculate linear regression (default: p <= 0.05)             |
| --logistic | {0.05}  | Calculate logistic regression (default: p <= 0.05)           |

**Example**

**Command format:**

Trustone --role {role} --bfile {input file} --abfile {check point alignment file} --acount {sample number

Quantity alignment} [GWAS function command] [parameter]

**Logical regression**

Assuming that both parties perform joint regression analysis on the data, the selected filtering parameter is: p-valule < 0.05.

**192.168.31.210 : (Suppose there are 1000 samples)**

 ```bash
 trustone --role lead --bfile /data/docker_fate/fate/confs-10000/shared_dir/data/testA --abfile /data/docker_fate/fate/confs- 10000/shared_dir/data/testB –acount 2000 --logistic 0.05 
 ```

**192.168.31.211 : (Suppose there are 2000 samples)**

```bash
trustone --role main --bfile /data/docker_fate/fate/confs-9999/shared_dir/data/testB --abfile /data/docker_fate/fate/confs-9999/shared_dir/data/testA –acount 2000 -- logistic 0.05  
```

