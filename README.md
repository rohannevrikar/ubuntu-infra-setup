## Ubuntu miniature environment setup

### High Level Design

![image](https://user-images.githubusercontent.com/26645175/184925658-8bad90d3-afb1-4a35-acc0-5d904df0300c.png)

- Public: Public network/internet
- Load balancer: A load balancer host which accepts requests only on ports 60000:65000. It can be accessed via ssh from anywhere using keys (no host restriction).
- Two webservers: Not accessible via public network, and only accessible by the load balancer host on port 80. They can be accessed via ssh only by the load balancer host. 
- Nagios server: The server used for monitoring other hosts/services. 

ssh key authentication is enabled in all the servers. 


### Setting up webserver

Used [nginx](https://www.nginx.com) as webserver.

Reasons behind chosing nginx:

- Prior familiarity.
- Easy to install and manage configurations.
- Can be used as load balancer and reverse proxy, both of which are required going forward for other use cases.


Steps:

- nginx installation: `sudo apt install nginx`
- Created /var/www/html/index.html in both the servers; each service different content so that we can differentiate while load balancing.
- The web page should now be serving at hostIP:80.

Resources
- https://www.digitalocean.com/community/tutorials/how-to-install-nginx-on-ubuntu-20-04



### Setting up load balancer

Used [nginx load balancer](https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer/).

Reasons behind chosing nginx as LB:

- Reusability of existing tool/framework and/or familiarity.
- It supports sticky balancing.
- Easy to configure backend pool and manage it.
- Easy for testing scenarios around availability of servers using configuration.

Steps:

- Updated LB nginx default config at [/etc/nginx/sites-available/default](https://github.com/rohannevrikar/ubuntu-infra-setup/blob/main/lb/etc/nginx/sites-available/default.conf).
- Created upstream server pool.
- Specified [ip_hash](http://nginx.org/en/docs/http/ngx_http_upstream_module.html#ip_hash) to configure "sticky" load balancing behaviour. With this, all the requests from the same client will be forwarded to the same host. 
- Specified ports to listen to in server block (60000:65000). As no ports are mentioned in `proxy_pass`, requests to other servers would be passed to port 80.
- Set `X-Forwarded-For` header to `$proxy_add_x_forwarded_for` for appending LB IP to the client IP in the request and send it to other servers. This way, client IP and LB IP will be forwarded to other servers.

Resources

- https://github.com/jay-nginx/load-balancing
- http://nginx.org/en/docs/http/load_balancing.html
- https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/

### Setting up Nagios monitoring

Nagios Core is required to be installed on the monitoring server, and NRPE is required to be installed on remote hosts which are to be monitored. NRPE helps to execute Nagios plugins on remote hosts. 

Steps:


- Host: Installed Nagios Core on monitoring server. 
- Host: Installed Nagios plugins.
- Host: Installed check_nrpe plugin.
- Remote host: Installed Nagios plugins.
- Remote host: Installed NRPE daemon.

Resources: 

- https://support.nagios.com/kb/article/nagios-core-installing-nagios-core-from-source-96.html#Ubuntu
- https://assets.nagios.com/downloads/nagioscore/docs/nrpe/NRPE.pdf


### User and authentication

A user needs to be created on all the boxes with sudo access. The user should be able to get in these servers using ssh keys. 

Steps (for each server):

- Created a user using `sudo adduser <username>`
- Granted sudo access: `sudo adduser <username> sudo`
- Copy public ssh keys to /home/username/.ssh/authorized_keys. Keys in authorized_keys file are the ones which will be authenticated against the private key at the time of ssh. 
- ssh from username@hostIP using private key. It should succeed. 

Resources:

- https://www.digitalocean.com/community/tutorials/how-to-configure-ssh-key-based-authentication-on-a-linux-server


### Network restrictions

Used `ufw` (Uncomplicated Firewall) to restrict/manage network related configuration.

Steps:

- For webservers, created the following rules: `sudo ufw allow from <LB-IP> to any port 22` - SSH from LB , `sudo ufw allow from <LB-IP> to any port 80` - Web traffic from LB and `sudo ufw allow from <Monitoring server> to any port 5666` - For Nagios to monitor this host.
- For load balancer, created the following rules: `sudo ufw allow 22` and `sudo ufw allow 60000:65000/tcp` to let SSH and HTTP traffic from anywhere.

The rules which are created will satisfy the following:

- Allow only LB server public ssh access
- From LB, webservers could be accessed via ssh
- Block all other unused/unnecessary ports
- Port 80 of webservers is only exposed for the LB to access
- Port 22 of webservers is only exposed for the LB to access


### Challenges

- Not able to make monitoring work as expected. Installed required things on respective hosts, but check_nrpe command fails with error `Connection reset by peer`. NRPE and Nagios are installed properly. 
- nginx throws a limitations of worker_connections if port range is more than 786, as it is the default value. Increased the number of worker_connections to match the number of open ports required, but it causes issues with server mostly because of memory. Ref: https://stackoverflow.com/questions/28265717/worker-connections-are-not-enough

### Scope for improvements:

- Enforcing SSL
