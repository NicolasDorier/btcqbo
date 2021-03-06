<h1>Quickbooks Online Connector for BTCPay Server</h1>

Copyright (C) 2018 Jeff Vandrew Jr

<h2>Introduction</h2>

Quickbooks Online (QBO) is currently the most popular small business bookkeeping solution, and is often used as an online invoicing solution. When customers receive an online invoice from a business using QBO, they currently have the option to pay electronically using credit card or ACH (via integrated Intuit Payments) or Bitcoin (via Intuit PayByCoin, which unfortunately only supports Bitpay and Coinbase). This software extends Quickbooks Online to take payments through BTCPay server. It is self-hosted and does not rely on any third party.

When this plugin is installed, customers choosing to pay a QBO invoice using BTCPay automatically have a BTCPay invoice generated with the customer's data pre-filled, and payments to the BTCPay invoice are autmatically recorded in QBO. Before generating the invoice, the software verifies that the email and invoice number match to prevent against customer typos.

<h2>Known Issues</h2>

Whenever you do an update on your BTCPay server installation or otherwise add or remove a Docker container to the network, you must repeat steps 7-11 in Part 2 to reconfigure nginx. A solution to this issue is being worked on.

Unfortunately the Quickbooks API does not allow emailing customer receipts. BTCPay does not support this either, leaving the feature to merchant software. Assuming there are no plans for BTCPay to support this feature, I will implement this in the plugin in the near future.

<h2>Notes</h2>

All payments made through BTCPay will be recorded in QBO in an "Other Current Asset" account called "Bitcoin." They are recorded at USD value as of the date the invoice was paid. This is not a bug; it is intentional behavior. The USD value on the payment date is the amount of taxable income recognized as well as the tax basis for a future sale of BTC under US Tax law, so the BTC is recorded in QBO accordingly. The information herein is educational only and is not tax advice; consult your tax professional.

Payments will not record in QBO until the invoice status in BTCPay is "confirmed." Payments are considered "confirmed" based on your BTCPay settings. The default is one on-chain confirmation.

<h2>Installation</h2>

Below are installation instructions for Dockerized deployment on a BTCPay Server instance which was set up via the one-click install referenced by the BTCPay team in their docs (LunaNode). More technical users can adapt these instructions for other setups.

<h3>Part 1: Obtain Intuit Keys</h3>

1. Head to https://developer.intuit.com and log in using the same login you normally use for Quickbooks Online. Your account will then be granted developer privileges.

2. After logging in, clock on "My Apps" and create a new app. The title is irrelevant.

3. After the "app" is created, click "Keys". There will be sandbox and production keys. To obtain the production keys, Intuit will require you to fully fill out your developer profile. Intuit will also require links to your privacy policy and EULA; these are largely irrelevant since your own business will the only "user" of your app. If you don't have links to a EULA and privacy policy, you may choose to use these links https://raw.githubusercontent.com/JeffVandrewJr/btcqbo/master/privacy-sample and https://raw.githubusercontent.com/JeffVandrewJr/btcqbo/master/eula-sample. These are provided for educational purposes, and consult with your own attorney if you have questions. 

4. On the Intuit Developer site, underneath your Intuit "production" keys, add "https://btcpay.example.com/btcqbo/qbologged" as a redirect URI, replacing btcpay.example.com with the domain where your BTCPay instance is hosted. Ensure you're doing this in the "production" (not sandbox) area of the page.

<h3>Part 2: Install BTCQBO</h3>

1. Log into your BTCPay LunaNode VPS using SSH. 

2. Using git, clone this repository to a local directory:
```
$ git clone https://github.com/JeffVandrewJr/btcqbo
```

3. Create an .env file by running `$ cp env.sample .env` in the btcqbo directory. Then, using the text editor of your choice, open the .env (example using nano as a text editor: `$ nano .env`). Be sure to enter your "client ID" and "client secret" from the keys tab on the Intuit Developer site. Also change the callback URL to the URL you chose in the last step of Part 1. Finally, change the BTCPay server URL to the URL of your BTCPay instance. After you're done, save the .env file and exit.

4. Run a Redis container:
```
$ sudo docker run --name redis --network=generated_default -d redis:latest
```

5. Build a btcqbo image (include the trailing period):
```
sudo docker build -t btcqbo .
```

5. Run a btcqbo container:
```
$ sudo docker run -d -p 8001:8001 --name btcqbo -e REDIS_URL=redis://redis:6379/0 --network=generated_default btcqbo:latest
```

6. Run an rq-worker container:
```
$ sudo docker run -d --name rq-worker -e REDIS_URL=redis://redis:6379/0 --network=generated_default --entrypoint "/usr/local/bin/rq" btcqbo:latest worker -u redis://redis:6379/0 btcqbo
```

7. Make a copy of the nginx default.conf out of its 
container (don't forget the trailing period): 
```
$ sudo docker cp nginx:/etc/nginx/conf.d/default.conf .
```

8. Open the default.conf you just copied in the text editor of your choice (example: `$ nano default.conf`) Just before the final closing curly brace in the file, add this code:
```
location /btcqbo/ {
proxy_pass http://btcqbo:8001;
proxy_redirect off;
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
 proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
}
[Final curly brace of the file is on this line.]
```

9. Copy the default.conf file back into the nginx Docker container: 
```
$ sudo docker cp default.conf nginx:/etc/nginx/conf.d/default.conf
```

10. Restart nginx by running the following three commands in succession (the final exit command is critical to avoid corrupting your nginx container):
```
$ sudo docker exec -it nginx /bin/bash
# service nginx reload
# exit
```

11. Set your username and password for the web interface by running the following three commands in succession (the final exit command is critical to avoid corrupting your container):
```
$ sudo docker exec -it btcqbo /bin/bash
# python3 cli.py setlogin
[follow the prompts]
# exit
```
You can run this command sequence to reset your login in the future if you forget it.

<h3>Part 3: Sync with Intuit & BTCPay</h3>

1. From a web browser, visit https://btcpay.example.com/btcqbo/authqbo, replacing btcqbo.example.com with your domain. Login with the username and password that you set above.

2. Follow the steps to sync to Inuit.

3. Go log into your BTCPay server, click on your store, and then hit settings. From the settings menu, create an authorization token. Once the token is created, BTCPay will provide a pairing code.

4. From a web browser, visit https://btcpay.example.com/btcqbo/authbtc, replacing btcqbo.example.com with your domain. Enter the pairing code from the step above, and submit.

<h3>Part 4: The Public Facing Payment Portal</h3>

These instructions assume your business' public facing page is a Wordpress site. 

1. Create a new page on your Wordpress site. Title it "Make a Bitcoin Payment". Set the URL so to something short, like example.com/pay.

2. Paste the code below into the body:
```
<form method="POST" action="https://btcpay.example.com/btcqbo/verify">
USD Amount:
  <input type="text" name="amount" />
Email Address:
  <input type="text" name="email" />
Invoice Number:
  <input type="text" name="orderId" />
  <input type="hidden" name="notificationUrl" value="https://btcpay.example.com/btcqbo/api/v1/payment" />
  <input type="hidden" name="redirectUrl" value="https://example.com" />
  <button type="submit">Pay now</button>
</form>
```

4. In the code, change btcpay.example.com to your appropriate domain, and set the redirect URL of your choice. Save the page in Wordpress; due to the magic of CSS it should automatically be styled to match your site.

5. In Quickbooks Online, edit your outgoing email template for invoicing with a concluding paragraph like this one:
```
"Click "Review and Pay below to pay via ACH or Credit Card, or click https://example.com/pay to pay via Bitcoin.
```

<h2>Troubleshooting</h2>

If QBO becomes unsynced, from the btcqbo directory try running:
```
$ sudo docker exec -it btcqbo /bin/bash
# python3 cli.py refresh
# exit
```
If the screen prints a bunch of JSON data, you've successfully resynced. If not, you may have to reauthorize from the web interface.

If you are familiar with RQ, you can view the RQ dashboard at https://btcpay.example.com/btcqbo/rq (replacing with your own domain). Access will be disabled if you're not logged into the web interface, so if you haven't previously logged in during a given session, head to https://btcpay.example.com/btcqbo/index to log in, then head to https://btcpay.example.com/btcqbo/rq. You must also ensure that RQ_ACCESS is set to `True` in your .env file.

Whenever you do an update on your BTCPay server installation or otherwise add or remove a Docker container to the network, you must repeat steps 7-11 in Part 2 to reconfigure nginx. A solution to this issue is being worked on.
