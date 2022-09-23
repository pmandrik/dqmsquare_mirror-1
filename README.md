
#### DQM^2 Mirror Page
This is a system to grab the information about DQM jobs from DQM^2 site, 
parse it, removing sensitive information, and show selections outside the P5 are.

The architecture is sequential:
* dqmsquare_robber.py - uses firefox webdriver (installed on the same pc) & selenium library to load DQM^2 page and execute JavaScript and save a copy to local folder. It is also used to click on *log* button in order to get the logs of the jobs withing the page, click on *graph* button in order to get the link to the graph-images and then save them as separate files into the tmp folder  
* dqmsquare_robber_oldruns.py - similare to dqmsquare_robber.py but click on *run number* buttons in order to switch to old runs and grab them too as separate files  
* dqmsquare_parser.py - parses the files made by dqmsquare_robber.py using based on BeautifulSoupin order to extract job status information and logs, lumis and run information and put it into html files. The parser is also remove tmp files older than TMP_FILES_LIFETIME.  
* dqmsquare_server.py - simple server to show html files made by dqmsquare_parser.py. Also host Control Room and other servises.  
* dqmsquare_cfg.py - for CFG and common code, run it to produce default .cfg file  
The work is periodic: dqmsquare_robber.py retrieves the information every X seconds, 
dqmsquare_parser.py tries to produce new html files every Y seconds,
JS at dqmsquare_server.py tries to update the content of the page using html files every Z seconds.

Other scripts/files:
* dqmsquare_deploy.sh - to download some extra software and run PyInstaller. We are using PyInstaller to pack python together with extra libraries (not a firefox!) into single executable ignoring lack of the software at P5 machines.
* dqmsquare_mirror.spec - to specify the RPM properties. Check this file in order to change the files used for the installation and paths. 
* services/dqmsquare_mirror@.service - configuration of daemons for DQM^2 Mirror services
* services/dqmsquare_mirror_wrapper.sh - used in order to define the working folder and execute dqmsquare_robber, dqmsquare_parser or dqmsquare_server
* static/dqm_mirror_template.html - the only 

Folders:
* log - folder to put logs files
* tmp - folder to put output from dqmsquare_robber and dqmsquare_parser
The RPM post-install script will create this folders. They are also hardcoded in the dqmsquare_server.py.

#### Deployment with RPM, Python 2.7
Download repo to your local linux machine. 
Check dqmsquare_deploy.sh to download extra dependencies and create executables.
Several options:

1. For testing copy whole package manually to the P5 machine (fusermount works well for me).
   From the same folder run dqmsquare_server.py at server machine, dqmsquare_robber.py at machine with firefox, dqmsquare_parser.py at any machine..
2. .. or copy RPM created by dqmsquare_deploy.sh to the P5 machine and install.  
   sudo rpm -i dqmsquare_mirror-1.0.0-1.x86_64.rpm  
   sudo systemctl enable dqmsquare_mirror@robber.service dqmsquare_mirror@robber_oldruns.service dqmsquare_mirror@parser.service dqmsquare_mirror@server.service  
   sudo systemctl start dqmsquare_mirror@robber.service dqmsquare_mirror@robber_oldruns.service dqmsquare_mirror@parser.service dqmsquare_mirror@server.service  
   You can also install this locally with --prefix=PATH option.  

Tested with:  
* Python: 2.7.14  
* Platform: Linux-4.12.14-lp150.12.82-default-x86_64-with-glibc2.2.5  
* Bottle: 0.12.19  
* Geckodriver: 0.29.1  
* PyInstaller: 3.4  
For the creation of RPM:

* rpm-build  

#### Deployment with Docker, K8, Python 3.6
We are using cmsweb k8 cluster to host our pods: server, parser and two grabbers. They are packed into single Docker image.
Server available to world-wide-web through cmsweb frontiend proxy. Grabber also connected to P5 through cmsweb frontiend proxy.
Cmsweb frontiend is required by default autentification using cern grid certificate, we are using one provided by cmsweb team to k8 cluster.
Also, by default Firefox do not know which certificate to use with cmsweb.cern.ch. We define rules in the Firefox profile locally and then pack profile into Docker image.
The connection at P5 to DQM^2 is closed without autentification cookie defined in DQM^2 backend (fff_web.py). 
Cookie autentification value is transfered to k8 cluster using Opaque Secret defined in k8_secret.yaml. 
To store log and tmp we mount CephFS. Claim for CephFS is defined in k8_claim_testbed.yaml for testbed cluster. In production and preproduction cluster CephFS volume is created by cmsweb team.
Also, they are requested Docker image to be defined to use not root user. Docker source image is python:3.9, cmsweb images not work well with firefox & geckodriver.
In general, source image and selenium, firefox, geckodriver versions are carefully selected to be able to work together with available code.

1. docker build -t registry.cern.ch/cmsweb/dqmsquare_mirror:v1.1.0 dqmsquare_mirror  
   For testing locally:
   docker run --rm -h `hostname -f` -v local_config_with_certificates:/firefox_profile_path -i -t registry.cern.ch/cmsweb/dqmsquare_mirror:v1.1.0_pre2  
2. docker login registry.cern.ch   
3. docker push registry.cern.ch/cmsweb/dqmsquare_mirror:v1.1.0_pre2  

For testbed cmsweb k8 cluster:
* update k8 config yaml to use registry.cern.ch/cmsweb/dqmsquare_mirror:v1.1.0_pre2
* At lxplus 8:
```
  export KUBECONFIG=/afs/cern.ch/user/m/mimran/public/cmsweb-k8s/config.cmsweb-test4
  kubectl apply -f k8_claim_testbed.yaml (once if PVC not available)
  kubectl apply -f k8_config_testbed.yaml
```
to login into a pod :   
```
  kubectl exec -it dqm-square-mirror-robber-74d5458fb-924j2 bash -n dqm 
```
While Service claim with port definition is avalable in testbed yaml maifest file it is not supported by cmsweb.

For preproduction cmsweb k8 cluster:
* get config from https://cms-http-group.docs.cern.ch/k8s_cluster/cmsweb_testbed_cluster_doc/ 
   and follow https://cms-http-group.docs.cern.ch/k8s_cluster/deploy-srv/ for deployment.
   We store yaml manifests at https://github.com/dmwm/CMSKubernetes:  
```
  export OS_TOKEN=$(openstack token issue -c id -f value)
  export KUBECONFIG=$PWD/config.cmsweb-testbed
  cd CMSKubernetes/kubernetes/cmsweb
  ./scripts/deploy-srv.sh dqmsquare v1.1.0_pre23 preprod
```
Also create Secret if it is not created before or edit it:
```
kubectl apply -f k8_secret.yaml -n dqm
kubectl edit secrets dqmsecret -n dqm
```
Secret value inside dqmsecret need to be in base64 format:
```
echo -n 'SECRET' | base64
```
To be able to connect DQM^2 at P5, the secrets at DQM^2 Mirror need to match secrets at DQM^2. 

Deployment to the production cmsweb is similare, follow:
https://cms-http-group.docs.cern.ch/k8s_cluster/cmsweb_production_cluster_doc/
```
wget https://cernbox.cern.ch/index.php/s/gLNiHYaGF8QbPrO/download -O config.cmsweb-k8s-services-prod
export KUBECONFIG=$PWD/config.cmsweb-k8s-services-prod
export OS_TOKEN=$(openstack token issue -c id -f value)
./scripts/deploy-srv.sh dqmsquare v1.1.0 prod
```

Tested with:  
* Python: 3.6  
* Bottle: 0.12.19  
* Geckodriver: 0.30.0  
* beautifulsoup4==4.10.0  
* selenium==3.141.0  
* Firefox 91.2.0esr  

#### DQM^2 Mirror Control Room (CR)
CR is a place to simplify the operations of DQM servises. It send requests to DQM^2 on one of the playback machine at P5 where requests are executed.
Secrets and versions of DQM^2 need to match DQM^2 Mirror requirements, please check for more info https://github.com/cms-DQM/fff_dqmtools

##### Scripts
1. dqmsquare_cert.sh imports .pem certificates provided by cmsweb k8 cluster into .p12 format and then into NSS sql DB used by firefox without master password.

#### Usefull extras
* Bottle built-in default server is not for a heavy server load, just for 3-5 shifters
* Number of logs created by dqmsquare_robber/dqmsquare_robber_oldruns/dqmsquare_parser/dqmsquare_server is limited by TimedRotatingFileHandler
* dqmsquare_robber.py spawn lot of firefox subprocesses. In case of the dqmsquare_robber process is killed they may persist, requiring the manual killing to free the resources.
*  to build  
 ./dqmsquare_deploy.sh build
* at p5 for installation, for example  
  sudo rpm -e dqmsquare_mirror; sudo rpm -i /nfshome0/pmandrik/dqmsquare_mirror-1.0.0-1.x86_64.rpm


   


