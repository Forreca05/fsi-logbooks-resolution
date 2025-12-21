# Public-Key Infrastructure LAB

## Setup

### Container Setup and Commands

Inside the *Labsetup* directory, the environment was built using the configuration provided in the `docker-compose.yml` file.

The following commands were executed:

```
dcbuild
dcup
```
To verify if the setup was correct, we used "dockps" and "docksh" and obtained this result:

![setup-containers](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/main/Semana%20%2311/Images/setup-containers.png?ref_type=heads)

### DNS Setup

Here, we edited the `/etc/hosts` to contain the following entries:

![setup-etc-hosts](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/main/Semana%20%2311/Images/setup-etchosts.png?ref_type=heads)

with www.guilherme2025.com and www.bank32.com present, we tested the connection by pinging the first entry:

![setup-ping](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/main/Semana%20%2311/Images/setup-ping.png?ref_type=heads)

## Task 1 - Becoming a Certificate Authority (CA)

The goal of this task is to establish a Root Certificate Authority (CA) by creating a self-signed certificate. This authority acts as the "Root of Trust" for the entire PKI system, allowing us to sign and validate other certificates.

Before generating cryptographic material, the environment must be prepared to function as a CA. This involves creating a specific directory structure for record-keeping and modifying the OpenSSL configuration to support multiple certificate issuances.

![Setup](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/main/Semana%20%2311/Images/Task1-setup.png?ref_type=heads)

We edited "openssl.cnf" to allow the creation of multiple certificates with the same subject name, which is necessary for the iterative nature of the lab tasks.

![unique_subject](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/main/Semana%20%2311/Images/Task1-unique_subject.png?ref_type=heads)

After this setup, using OpenSSL, we generated a 4096-bit RSA private key and a self-signed Root CA certificate.

![generate-key](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/main/Semana%20%2311/Images/task1-Generatekey.png?ref_type=heads)

Then, following the commands given by the guide, we were able to gather the information needed to answer the questions asked in the end of task 1.

*What part of the certificate indicates this is a CA’s certificate?*

The certificate is confirmed as a CA certificate through the X509v3 Basic Constraints extension, as the field states CA:TRUE.

![CA-TRUE](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/main/Semana%20%2311/Images/Task1-CA-True.png?ref_type=heads)

*What part of the certificate indicates this is a self-signed certificate?*

A certificate is self-signed if the Issuer and the Subject are identical and both fields are set to "CN = www.modelCA.com, O = Model CA LTD., C = US"

![Self-Identifying](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/main/Semana%20%2311/Images/task-1-self-identifying.png?ref_type=heads)

*In the RSA algorithm, we have a public exponent e, a private exponent d, a modulus n, and two secret
numbers p and q, such that n = pq. Please identify the values for these elements in your certificate
and key files.*

![public-exponent](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/main/Semana%20%2311/Images/task1-publicExponent.png?ref_type=heads)
![private-exponent](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/main/Semana%20%2311/Images/task1-privateExponent.png?ref_type=heads)
![modulus](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/main/Semana%20%2311/Images/task1-modulus.png?ref_type=heads)
![prime1-prime2](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/main/Semana%20%2311/Images/task1-prime1prime2.png?ref_type=heads)

## Task 2 - Generating a Certificate Request for Your Web Server

The goal of this task is to simulate a website owner generating a Certificate Signing Request (CSR). This request contains the website's public key and identity information, which is then sent to a trusted Certificate Authority to be digitally signed.

To obtain a trusted certificate, we first had to generate a private key for the server and a corresponding CSR. We used the domain www.guilherme2025.com as our identity and included the two asked alternative names to the certificate signing request.

![key-generation](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/main/Semana%20%2311/Images/task2-keygen.png?ref_type=heads)

Then, using "openssl req -in guilherme.csr -text -noout" we were able to get the certificate, with the alternative names:

![Certificate](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/main/Semana%20%2311/Images/task2-certificate.png?ref_type=heads)

## Task 3 - Generating a Certificate for your server

The objective of this task is to act as the Certificate Authority (CA) and use the Root CA's private key (ca.key) to sign the Certificate Signing Request (CSR) created in Task 2. This process generates a valid digital certificate (guilherme.crt) that binds the website's identity and public key with the trusted signature of the CA.

To transform the CSR into a trusted certificate, the CA must process the request using its configuration and master key.

Firstly, we uncommented the line "copy extensions = copy"

![Copy_extensions](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/main/Semana%20%2311/Images/task2-certificate.png?ref_type=heads)

Using the OpenSSL ca command, we signed the guilherme.csr file.

![csr](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/main/Semana%20%2311/Images/task3-certificate.png?ref_type=heads)

Finally, we used "openssl x509 -in guilherme.crt -text -noout" to verify that the alternative names were present in the certificate

![Alternative-names](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/main/Semana%20%2311/Images/task3-names.png?ref_type=heads)

## Task 4 - Deploying Certificate in an Apache-Based HTTPS Website

The goal of this task is to configure an Apache web server with the signed certificate from Task 3 and verify that a client browser trusts the connection. This demonstrates a complete, functional PKI "Chain of Trust" in a real-world scenario.

Firstly, we moved the files "guilherme.key", "guilherme.crt" and "guilherme.crs" into the lab's shared volume directory:

![volumes](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/main/Semana%20%2311/Images/task4-volumes.png?ref_type=heads)

Then we got inside the image given to us by the labsetup, using "docksh e9"

![image](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/main/Semana%20%2311/Images/setup-containers.png?ref_type=heads)

From here, we ran ```nano /etc/apache2/sites-available/guilherme2025_apache_ssl.conf``` to create a file for our server. It contains the following:

![firstconfig](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/main/Semana%20%2311/Images/task4-apacheconfig.png?ref_type=heads)

For now, we'll use the same html as bank32, so we keep the ```DocumentRoot /var/www/bank32```. We complete the setup with the commands given by the guide, just to double check, and then use "service apache2 start"

![apacheCommands](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/main/Semana%20%2311/Images/task4-startapache.png?ref_type=heads)

Then, we visit the website <https://www.guilherme2025.com> and we are prompted with a warning that asks us whether we want to continue. After saying yes, we get this:

![sitewithbank](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/main/Semana%20%2311/Images/task4-site-with-bank.png?ref_type=heads)

Now, we'll change the html of our website. For this, we will change the ``guilherme2025_apache_ssl.conf`` so that the Document Root is now "/var/www/guilherme2025"

![configchanges](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/main/Semana%20%2311/Images/task4-configchanges.png?ref_type=heads)

Now, we only need to create the new path and index file

``` bash
mkdir /var/www/guilherme2025
nano /var/www/guilherme2025/index.html
    <h1>Different form Bank32</h1>
    <p>FSI 2025</p>
```

Before visiting the site, the Root CA certificate (ca.crt) was imported into the Firefox browser. We checked the box "This certificate can identify websites" to authorize the browser to trust certificates signed by our "Model CA".

![CA-trust](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/main/Semana%20%2311/Images/task4-catrust.png?ref_type=heads)

So, when we access <www.guilherme2025.com> we get the "Different from Bank32" and the secure connection to the site (we are not prompted with the warning anymore)

![siteguilherme](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/main/Semana%20%2311/Images/task4-site-with-guilherme.png?ref_type=heads)

To end this task, if we look at the certificate, we will see the organization is now "Guilherme Bank", instead of bank32, and the alternative DNS Names appear sucessfully

![cetificate](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/main/Semana%20%2311/Images/task4-site-certificate.png?ref_type=heads)

## Task 5 - Launching a Man-In-The-Middle Attack

The goal of this task is to observe how the browser's identity verification process protects users from a Man-In-The-Middle (MITM) attack. Specifically, we demonstrate that even if an attacker successfully redirects traffic (DNS poisoning), the browser will block the connection if the presented certificate does not match the requested domain name.

For the first step, we created a new _apache_ssl.conf file,for the website we want, in our case, youtube. So, we use ``nano /etc/apache2/sites-available/youtube_apache_ssl.conf`` to create the file:

![youtubeconfig](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/main/Semana%20%2311/Images/task5-ytconfig.png?ref_type=heads)

We then changed out "etc/hosts" and added ``10.9.0.80   www.youtube.com``. This way, when we open <www.youtube.com>, we will be redirected to our own html, instead of the actual public website.

![etchosts](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/main/Semana%20%2311/Images/task5-modifyingetchosts.png?ref_type=heads)

And so, when we try to go to youtube, we get prompted with the warning once more (connection is blocked), and if we go through with connection to the website, we get this:

![youtubevisit](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/main/Semana%20%2311/Images/task5-ytvisit.png?ref_type=heads)

We are redirected to our html, and the connection is marked as not secure (as we can see from the padlock with the red stripe).

## Task 6 -  Launching a Man-In-The-Middle Attack with a Compromised CA

For this final task, we followed the same setup as Task 5, only now for Google.

We created a google_apache_ssl.conf (and added ``10.9.0.80   www.google.com`` to "etc/hosts"):

![googleapache](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/main/Semana%20%2311/Images/task6-apache.png?ref_type=heads)

These "malicious.crt" and "malicious.key" are the ones we generated and signed using our compromised ca.key. Since we possess the trusted key, the resulting certificate is cryptographically indistinguishable from a real one to the browser.

![malicious-key](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/main/Semana%20%2311/Images/task6-maliciouskey.png?ref_type=heads)
![maliciouscsr](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/main/Semana%20%2311/Images/task6-certificate.png?ref_type=heads)

So, after this setup, when we search for <www.google.com> we are not prompted with any warning whatsoever, and we are still redirected to our html, with a "secure connection" (as we can se from the secure padlock).

![Final](https://gitlab.up.pt/class/fsi/2526/t17/t17-group04/-/raw/main/Semana%20%2311/Images/task6-final.png?ref_type=heads)

Unlike Task 5, where the browser protects the user from a name mismatch, Task 6 succeeds because we used a compromised key to create a certificate that is cryptographically valid AND identically named.

