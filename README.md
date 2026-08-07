# GenePatternGistic_ncl

Running GISTIC 2.0 module from the [GenePatternModuleCatalog](https://github.com/genepattern/GenePatternModuleCatalog/) on a Self-Hosted [GenePattern Server](https://github.com/genepattern/genepattern-server) (No Docker or root requirements):  
This documents how GISTIC 2.0 was set up and run on our shared Linux server (`titania`), without Docker and without sudo/root access, after the public GenePattern server and Broad's FTP support-file server both became unavailable.

### Environment
* Ubuntu Linux server, no sudo access
* Conda environment named `GenePattern`
* GenePattern server installed to local directory
* Access via SSH; GenePattern web UI reachable directly at `http://titania:8080/gp` or `http://{server_name}:8080/gp` or `http://{ip_address}:8080/gp`.

### 1. Java setup

GenePattern requires Java 8 or 9 - create a conda environment and manually set Java version.

```
conda create --name GenePattern  
conda activate GenePattern  
conda install -c conda-forge openjdk=8
```

Verify with `java -version` before proceeding - all subsequent steps assume this env is active.

### 2. Installing GenePattern server (headless)
Download the latest release (GPserver.bin, build b439) from the [genepattern-server releases page](https://github.com/genepattern/genepattern-server/releases) and run a silent, headless install:
If this download link becomes stale, please visit `https://github.com/genepattern/genepattern-server/releases/` to find a suitable alternative.
```
mkdir -p ~/genepattern-install && cd ~/genepattern-install
wget https://github.com/genepattern/genepattern-server/releases/download/v3.9_010926_b439/GPserver.bin
chmod u+x GPserver.bin
cat > installer.properties << EOF
USER_INSTALL_DIR=$HOME/GenePattern
INSTALLER_UI=silent
INSTALLER_TEMP_DIR=/tmp
IATEMPDIR=/tmp
EOF

export PS1=">"   # workaround for a known Ubuntu/LAX prompt-parsing bug
./GPserver.bin -f installer.properties
```
Registration workaround - the old registration server no longer exists, so this property must be added manually or the server won't start:
`echo "registeredServer=true" >> ~/GenePattern/resources/genepattern.properties`


#### Convenience scripts
Convenience `start_server.sh` / `stop_server.sh` scripts should be added to the install root for repeat use to avoid repeatedly killing/restarting the Java process by hand. First change directory, `cd ~/GenePattern` then:  
Create `start_server.sh`:
```
cat > ~/GenePattern/start_server.sh << 'EOF'
#!/bin/bash
# Starts the GenePattern server
# NOTE: activate the GenePattern conda env (Java 8) before running this

set -e

export PS1=">"

cd "$HOME/GenePattern"
nohup ./StartGenePatternServer > "$HOME/GenePattern/logs/server_stdout.log" 2>&1 &
echo "Started GenePattern server with PID $!"
EOF
```
Create `stop_server.sh`:
```
cat > ~/GenePattern/stop_server.sh << 'EOF'
#!/bin/bash
# Stops the GenePattern server by finding and killing its java process

PID=$(ps aux | grep '[c]om.zerog.lax.LAX' | awk '{print $2}')

if [ -z "$PID" ]; then
    echo "No GenePattern server process found."
    exit 0
fi

echo "Stopping GenePattern server (PID $PID)..."
kill "$PID"

# Wait up to 15 seconds for a clean shutdown
for i in {1..15}; do
    if ! ps -p "$PID" > /dev/null 2>&1; then
        echo "Stopped."
        exit 0
    fi
    sleep 1
done

echo "Process still running after 15s, forcing kill..."
kill -9 "$PID"
echo "Stopped (forced)."
EOF

chmod +x ~/GenePattern/stop_server.sh
```
Make both scripts executable:  
`chmod +x ~/GenePattern/start_server.sh`  
`chmod +x ~/GenePattern/stop_server.sh`

Usage:
```
conda activate GenePattern
~/GenePattern/start_server.sh
~/GenePattern/stop_server.sh
```


### 3. Installing the GISTIC_2.0 module

Pull the module directly from Broad's [GenePatternModuleCatalog repo](https://github.com/genepattern/GenePatternModuleCatalog) (public GenePattern module repository is down).

Two versions exist here:
* 6.15.28 - Dockerized. Not usable without Docker.
* 6.14 - Uses a legacy MATLAB Compiler Runtime (MCR) wrapper (<run-with-env> -u matlab-mcr/2014a ...) instead of Docker. This is the version to use without Docker installed.

#### 3a. Enable the run-with-env wrapper system
```
cat ~/GenePattern/resources/wrapper_scripts/wrapper.properties \
  >> ~/GenePattern/resources/custom.properties
```

#### 3b. Install MATLAB Runtime R2014a (MCR v8.3), no root required
```
mkdir -p ~/mcr_install && cd ~/mcr_install
wget https://ssd.mathworks.com/supportfiles/downloads/R2014a/deployment_files/R2014a/installers/glnxa64/MCR_R2014a_glnxa64_installer.zip
unzip MCR_R2014a_glnxa64_installer.zip -d installer
cd installer
./install -mode silent -agreeToLicense yes -destinationFolder ~/mcr
```

#### 3c. Site-customization script (`env-custom.sh`)
GenePattern's `run-with-env` mechanism does nothing by default - it must be told how to set up each named runtime environment. Create:
`~/GenePattern/resources/wrapper_scripts/env-custom.sh`

```
#!/usr/bin/env bash
function initEnv() {
    local module="$1"
    case "$module" in
        matlab-mcr/2014a)
            export MCR_ROOT="$HOME/mcr/v83"
            export LD_LIBRARY_PATH="$HOME/local_libs:${MCR_ROOT}/runtime/glnxa64:${MCR_ROOT}/bin/glnxa64:${MCR_ROOT}/sys/os/glnxa64:${MCR_ROOT}/sys/opengl/lib/glnxa64:${LD_LIBRARY_PATH}"
            export XAPPLRESDIR="${MCR_ROOT}/X11/app-defaults"
            ;;
        *)
            :
            ;;
    esac
}
```

#### 3d. Missing old shared libraries

The compiled GISTIC binary was built against ncurses libraries Ubuntu no longer ships. Extracted directly from official .deb packages into `~/local_libs` (already on LD_LIBRARY_PATH via `env-custom.sh` above):

```
mkdir -p ~/local_libs
cd /tmp

wget http://archive.ubuntu.com/ubuntu/pool/universe/n/ncurses/libncurses5_6.3-2_amd64.deb
dpkg-deb -x libncurses5_6.3-2_amd64.deb ncurses_extracted
cp ncurses_extracted/lib/x86_64-linux-gnu/libncurses.so.5.9 ~/local_libs/libncurses.so.5

wget http://archive.ubuntu.com/ubuntu/pool/universe/n/ncurses/libtinfo5_6.3-2_amd64.deb
dpkg-deb -x libtinfo5_6.3-2_amd64.deb tinfo_extracted
cp tinfo_extracted/lib/x86_64-linux-gnu/libtinfo.so.5.9 ~/local_libs/libtinfo.so.5
```


### 4. Server config fix

#### 4a. Create custom config .yaml file 
GenePattern reads a single config file: `config_custom.yaml` if it exists, otherwise `config_default.yaml` - it does not merge the two. The correct approach is always to copy the full default file first:
```
cp ~/GenePattern/resources/config_default.yaml \
   ~/GenePattern/resources/config_custom.yaml
```
#### 4b. Disable the global Docker command prefix

By default, GenePattern's LocalJobRunner executor silently prepends `<run-with-docker>` to every job's command line, regardless of the module's own manifest. This causes `Command not found: '/usr/local/bin/docker'` errors even for the non-Docker MCR module. Fix it by blanking the prefix:

```
python3 << 'PYEOF'
path = "$HOME/GenePattern/resources/config_custom.yaml".replace("$HOME", __import__("os").path.expanduser("~"))
with open(path) as f:
    content = f.read()

old = '            "job.commandPrefix": "<run-with-docker>"\n'
new = '            "job.commandPrefix": ""\n'

if old not in content:
    raise SystemExit("Target line not found — check indentation/spacing manually")

content = content.replace(old, new)

with open(path, "w") as f:
    f.write(content)

print("Done: job.commandPrefix cleared")
PYEOF
```

Note: this is a server-wide change. Any future module that does expect to run via Docker will need this re-enabled or overridden per-module.



### 5. Module manifest patch (final version: 6.14.3)

The `refgene.file` parameter in the stock module manifest is tied to a dead FTP dropdown with no usable upload option in this GenePattern version. Fixed by editing the module's manifest file to remove the dynamic-choice attributes entirely, turning it into a plain upload parameter (identical in behavior to `seg.file`/`markers.file`):

```
mkdir -p ~/gistic_patch && cd ~/gistic_patch
wget "https://github.com/genepattern/GenePatternModuleCatalog/raw/main/modules/GISTIC_2.0/broad.mit.edu%3Acancer.software.genepattern.module.analysis/00125/6.14/GISTIC_2.0.zip"
unzip GISTIC_2.0.zip -d unzipped
cd unzipped

sed -i '/^p1_choiceAllowCustom=/d; /^p1_choiceDir=/d; /^p1_choiceDirFilter=/d' manifest
sed -i 's/:00125:6.14$/:00125:6.14.3/' manifest
chmod +x gp_gistic2_from_seg

zip -r ../GISTIC_2.0_v6.14.3_final.zip doc.html gp_gistic2_from_seg manifest paramgroups.json
cd ..
```
This is basically repackaged as a new version, which can be installed as a module on the GenePattern GUI (as detailed in section 7).
Separately, the extracted binary lost its executable bit during zip/reinstall - fix it directly on the server like so:
`chmod +x ~/GenePattern/taskLib/GISTIC_2.0.6.14.3.*/gp_gistic2_from_seg`

### 6. Reference genome file & input file formatting

* Get the reference .mat files (hg19 for us) which are mirrored in Broad's own gistic2 GitHub repo - no need for the dead FTP server:  
  `wget https://github.com/broadinstitute/gistic2/raw/master/support/refgenefiles/hg19.UCSC.add_miR.140312.refgene.mat`
* Ensure the original seg file (from conumee) is tab-delimited with the following 6 columns only `"Sample"`,`"Chromosome"`,`"Start"`,`"End"`,`"Num_Probes"`,`"Seg.CN"`.
  The "chr" prefix needs to also be removed from the `"Chromosome"` column prior to use here.

### 7. Final working job configuration
* Activate the conda environment:
  `conda activate GenePattern`
* Start the local GenePattern server:
  `~/GenePattern/start_server.sh`
* Navigate to `http://titania:8080/gp` or `http://{server_name}:8080/gp` or `http://{ip_address}:8080/gp`.
* Ensure updated GISTIC2.0 module (`GISTIC_2.0_v6.14.3_final.zip`) is available on the GenePattern GUI. If not, install by hovering over Modules, select Modules & Pipelines, then select Install from Zip (ensure you install for all users).
* Navigate to GISTIC2.0 module via "Browse modules" and select it to be directed to the run page.
* Use hg19.UCSC.add_miR.140312.refgene.mat as the reference.
* Use pre-generated seg file (as per section 6).
* Run without markers file, using built-in pseudo-marker generation instead (maxspace parameter, left at default 10000).
* In settings, set gene.gistic to yes.
* Download the job files once completed from the job page.

### 8. Cleanup notes

Installer staging directories (~/mcr_install, /tmp/*_extracted, downloaded .deb/.zip files) are safe to delete post-install.
config_custom.yaml's job.commandPrefix override is server-wide - worth remembering if a future Docker-based module is installed on this server.
