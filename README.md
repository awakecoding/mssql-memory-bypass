# mssql-memory-bypass

Bypass the SQL Server for Linux minimum memory requirement (2GB):

> sqlservr: This program requires a machine with at least 2000 megabytes of memory.

**Why?** Because in memory-constrained lab environments, performance is secondary. SQL Server only checks for available RAM at startup without taking swap files into account. However, bypass this check at your own risk, and experiment to see if the performance remains acceptable for your intended use case.

**How?** By using the good old LD_PRELOAD trick to load a wrapper library, and fake the information returned by the sysinfo() call. The original wrapper library code is borrowed from [myssql_server_tiny](https://github.com/justin2004/mssql_server_tiny).

## Quick Start

Assuming [SQL Server is already installed](https://learn.microsoft.com/en-us/sql/linux/sql-server-linux-setup?view=sql-server-ver16), clone this repository, install the prebuilt fake_meminfo.so library, then modify the mssql-server service configuration to set the LD_PRELOAD environment variable:

```bash
git clone https://github.com/awakecoding/mssql-memory-bypass
cd mssql-memory-bypass
sudo cp fake_meminfo.so /opt/mssql/lib/fake_meminfo.so

sudo mkdir -p /etc/systemd/system/mssql-server.service.d
sudo tee /etc/systemd/system/mssql-server.service.d/override.conf > /dev/null <<EOF
[Service]
Environment="LD_PRELOAD=/opt/mssql/lib/fake_meminfo.so"
EOF

sudo systemctl daemon-reload
sudo systemctl restart mssql-server
```

Since SQL Server for Linux does not support ARM, the prebuilt shared library only targets x86_64. I've built it Ubuntu, hopefully it should work on other distributions.

## Building wrapper library from source

If you wish to build the wrapper library from source, just install cmake and build tooling first:

```bash
git clone https://github.com/awakecoding/mssql-memory-bypass
sudo apt install build-essential cmake
cmake .
make
sudo cp fake_meminfo.so /opt/mssql/lib/fake_meminfo.so
```

I am including a prebuilt wrapper library in this repository to avoid the need to rebuild it from source. The virtual machine where SQL Server is running probably doesn't have build tooling installed, so it's a pain to build it.

## Creating swap file

To ensure SQL Server has enough memory despite the very small amount of real memory available, create a new swap file:

```bash
sudo fallocate -l 8G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

You can also increase the size of your current swap file. 8GB should be good enough, but you can always go higher depending on your needs.

## Installing SQL Server for Linux

For SQL Server 2022, [I recommend using Ubuntu 22.04](https://learn.microsoft.com/en-us/sql/linux/quickstart-install-connect-ubuntu?view=sql-server-ver16&tabs=ubuntu2204):

```bash
curl https://packages.microsoft.com/keys/microsoft.asc | sudo tee /etc/apt/trusted.gpg.d/microsoft.asc
sudo add-apt-repository "$(wget -qO- https://packages.microsoft.com/config/ubuntu/$(lsb_release -rs)/mssql-server-2022.list)"
sudo apt-get update -y
sudo apt-get install -y mssql-server
```

You can then initialize the super admin password and accept the EULA:

```bash
sudo LD_PRELOAD='/opt/mssql/lib/fake_meminfo.so' MSSQL_PID='Express' MSSQL_SA_PASSWORD='SuperPass123!' /opt/mssql/bin/mssql-conf -n setup accept-eula
```

Don't forget to set LD_PRELOAD='/opt/mssql/lib/fake_meminfo.so' here as well to bypass the memory check!
