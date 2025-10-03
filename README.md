# A Middlebox for Verification of Encrypted FaaS Traffic

## Virtual machine deployment

These steps must be run in the `ETSI` folder for the TLMSP middlebox, and in the `DC` folder for the Delegated
Credentials middlebox.\
_\<mb>_ is _etsi_ for TLMSP and _mb_ for DC.

### Installation

1. Install [VirtualBox](https://www.virtualbox.org/wiki/Downloads)
2. Install [Vagrant](https://developer.hashicorp.com/vagrant/downloads)
3. `cd vm`
4. `vagrant up`, checking that all provisioning scripts run successfully
5. Open 3 terminal windows, and for each of _client_, _\<mb>_, _openfaas_ run `vagrant ssh {name}`. It can be preferable
   to have the terminals in this order, as to resemble the configuration of the physical network.
6. On _openfaas_: `cd external/hello-retail/kubernetes; sudo ./deploy.sh`

#### Notes

- While instructions for this won't be provided, the _openfaas_ VM built for TLMSP can be reused for the DC environment.\
The _client_ VM can also be reused (also from TLMSP to DC), either by copying the Go installation manually or by
directly using the `client` executable compiled from the _mb_ VM.
- The openfaas webui is available on the host on http://127.0.0.1:5001. The username is `admin`; to get the password,
  run on the server   
  `sudo kubectl get secret -n openfaas basic-auth -o jsonpath="{.data.basic-auth-password}" | base64 --decode`

### Network layout

- Client (_client_):
    - IP: `192.168.56.1`
    - Interface: `eth1`
- Middblebox (_\<mb>_, client side):
    - IP: `192.168.56.2`
    - Interface: `eth1`
- Middblebox (_\<mb>_, server side):
    - IP: `192.168.58.2`
    - Interface: `eth2`
- Server (_openfaas_):
    - IP: `192.168.58.1`
    - Interface: `eth1`

For more information check the `ip-*.sh` scripts

Check that the machines can ping each other by running `ping {IP} -I {interface}` and `ping {IP}` (the correct routes
are preconfigured so both commands should work).

## Bare-metal deployment

It is recommended to use Ubuntu 22.04, as this is the only tested OS.

No specific scripts are provided for bare-metal deployment, as the Vagrant scripts should work with a small amount of
modifications. Check the `Vagrantfile` for the selected deployment and run the corresponding scripts (`provision-XXX.sh`
for the first configuration and `ip-XXX.sh` after every reboot, using the correct interface names).\
Instead of using the `shared` and `external` folders, their respective sources can be used.

It is recommended to create only a single client, middlebox and server machine, supporting both TLMSP and DC (see [VM installation notes](#notes) for client and server, and for the middlebox execute the
provisioning scripts for both variants).

The tested network layout is the same as the VM one, with two ethernet cables connecting client-middlebox and
middlebox-server. A configuration with a switch could be used, but it was not tested.\
It was observed that the IP addresses sometimes get deleted after being set, `systemctl stop NetworkManager` resolves
this issue, and if wireless connection is needed `systemctl start NetworkManager` can be run without side effects after
all the machines' connections have been setup.

Note that, for testing, an active internet connection will be required for all machines (at the startup of the middlebox
executables, at the startup of openfaas, and to get a token on the client). Having an additional wireless or wired
connection is preferable, but if only one connection is available the various executables can be run (and stopped,
except for OpenFaaS) before changing the layout.

## Execution

To run the middlebox functionality, use the following commands

### Server (all variants)

OpenFaaS
```
sudo kubectl port-forward --address 0.0.0.0 -n openfaas svc/gateway 8080:8080 >/dev/null 2>/dev/null &
```
Running `curl 127.0.0.1:8080/function/init` should return no output

Alternative: plain Nginx
```
sudo apt install nginx
sudo systemctl start nginx
```

### DC

#### Server

No additional commands are required

#### Middlebox

- Generate a self-signed certificate (run with Cloudflare GO):
```
go run <cfgo-src-path>/src/crypto/tls/generate_cert.go -host 127.0.0.1 -allowDC
```
- If the delegated credential has not been generated already (dc.cred is not in DC/certs) (run with Cloudflare GO)
```
go run <cfgo-src-path>/src/crypto/tls/generate_delegated_credential.go -cert-path cert.pem -key-path key.pem -signature-scheme Ed25519 -duration 168h
```
NOTE: Remember that the DC expires after 7 days

```
cd ~/shared/Middlebox
# ./compile.sh if required
./middlebox
```

The terminal should then stop asking for input

#### Client

#### Getting an OAuth token
- Create an OAuth application for example, from [Google Cloud](https://console.cloud.google.com/auth/clients)
- Click "Create Client" -> Web application -> any name. As "Redirect URL" set `http://localhost/404`
- Copy Client ID and Client Secret, and double-check the  and paste them in the right place in the file `OAuthTokenGetter/secrets.js`
- With any web browser open the HTML page  `OAuthTokenGetter/get_code.html` and click "Get access code". Log in with your Google account
- From the page URL, copy the parameter "code" and paste it into the "Get refresh token" text box and click the button. A token should appear 
- Create a refresh_token.txt file in the same folder
- Paste the "refresh_token" parameter from the access token returned by Google
- The refresh_token.sh should always return a JSON token.
```

```

```
cd ~/shared/Middlebox
# ./compile.sh if required
```

To check that everything works, `./client 'https://<middlebox-ip>:<middlebox-port>/function/init'` should have no output, then proceed
to the Testing phase.

### TLMSP

#### Server

```
httpd -X
```

The terminal should then stop asking for input  
If after running httpd you can run more commands, then it failed to start. Check the logs in `~/tlmsp/install/var/logs/`

#### Middlebox

```
cd ~/shared/Middlebox
tlmsp-mb -c ~/shared/Configurations/randomization.ucl -t mbox1 -P
```

The terminal should then stop asking for input

#### Client

To check that everything
works, `curl -k --tlmsp /shared/Configurations/randomization.ucl 'https://192.168.58.1:4444/function/init'` should have no
output, then proceed to the Testing phase.
## Testing

Since the automatic script tests all the methods for TLMSP and DC, it is recommended to use a bare-metal deployment with
all functionalities.
```
cd PerformanceMeasuring
pip install -r requirements.txt
```

#### Manual testing

The `measure.py` script is available to run a single measurement. The correct middlebox executable must be run manually,
and optionally `httpd` on the server for TLMSP tests.

#### Automatic testing

The `automate.py` script takes care of starting the correct middlebox executables and `httpd` when needed, and runs all
the configured tests.\
The middlebox and server need to have an SSH server installed.\
For the first execution, edit `automate.py` with the correct paths and passwords.

### Results evaluation

The `plot.py` script creates plots from the saved results. Running it will give more information on its usage.

If LaTeX fonts are required for the output graphs, run:
```
sudo apt-get install dvipng texlive-latex-extra texlive-fonts-recommended cm-super
```
