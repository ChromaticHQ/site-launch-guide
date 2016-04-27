# Site Launch Guide {#site-launch-guide}
Below is a site launch checklist, with details on individual areas of interest to follow in the [appendix](#appendix). While some are Drupal specific, the majority would apply to most any site.

## Launch Checklist {#launch-checklist}
* Is the web server instance size [large enough](#infrastructure)?
* Is there a [load balancer](#infrastructure) in front of your web head(s)?
* Is [Jenkins](#automation) configured to automatically deploy your code, run cron, etc?
* Is [Redis](#redis) configured and enabled?
* Is a [CDN](#cdn) configured?
* Is the CDN serving HIT's?
* Is [Varnish](#varnish) serving HIT's?
* Is [New Relic configured](#new-relic)?
* Is the VirtualHost configured to redirect from www to the base url (or vice-versa)?
* Is [HTTPS](#https) enabled?
* Is Apache configured for [HTTP/2](#http2)?
* Is Google Analytics (or your analytics tool of choice) configured?
* Is robots.txt configured for production (ie. did you remove any changes that were made for development)?
* Is Drupal's [internal page cache](https://www.drupal.org/documentation/modules/internal_page_cache) enabled?
* Is the [Security Review](https://www.drupal.org/project/security_review) module installed and providing a clean report? 
* Do Drupal's settings.php & (if Drupal 8) services.yml files have the correct read-only permissions?
* Are all of the checks on Drupal's status report page reporting green?
* Are all development related modules disabled?
* Are errors configured to be suppressed?

## Appendix {#appendix}
### Infrastructure {#infrastructure}
Though we use a number of different hosting providers in practice, our standard is [Linode](https://www.linode.com/?r=05d012f41778d9c4d4e56f9f0b0f7c0394dc41a0). Specific hardware recommendations follow:

#### Web Server {#web-server}
Use at least a 4GB cloud instance. If you or the client are price sensitive and are considering opting for a smaller instance size to save money, I would argue that the billable time spent troubleshooting an underperforming server is easily much more expensive then paying for more power.

#### Load Balancer {#load-balancer}
A load balancer is essential when configuring a site with multiple web servers, but using a load balancer is preferable even in situations with only one web server. Having DNS point to a load balancer, instead of to the web server directly, will give you instantaneous control over where your traffic is routed. For example, if you need to replace your web server hardware, you can redirect traffic instantaneously as opposed to waiting for DNS to propagate. Additionally, a load balancer can add simplicity when configuring a site that uses [HTTPS](#https), as you can configure the appropriate certificates at the load balancer level as opposed to on all of the relevant web servers.

### Automation {#automation}
At a minimum, the following jobs should be configured in [Jenkins](https://jenkins.io):

* Automated deployments triggered from Github.
* Cron to be run at least once every 24 hours.
* A Drush cache clear job that can be run on-demand from the Jenkins UI.

### Performance {#performance}
#### New Relic {#new-relic}
Chromatic configures all web servers meant for production with New Relic. If the client does not already have a New Relic account, create one and obtain the license key. When configuring production boxes using Ansible, utilize the New Relic role in the playbook and provide the correct API key.

#### Redis {#redis}
[Redis](http://redis.io/) should be installed and configured for all production Drupal sites. Using Redis will improve database performance.

#### CDN {#cdn}
Putting a CDN in front of your site, provides many perfomance and security benefits. With the many low-cost and free options available, there is rarely a reason to not institute a CDN on every production site. We have had great success using [CloudFlare](https://www.cloudflare.com/).

_Note: CloudFlare [requires you to change your name servers](https://support.cloudflare.com/hc/en-us/articles/200172566-Why-do-I-have-to-change-my-DNS-settings-to-use-CloudFlare-) and use them for DNS configuration. These changes should be made at least 24 hours in advance of launch._

If this is a Drupal 7 site, be sure to add the following line to your production settings.php file:
```
/**
 * Remove "cookie" from Vary header to allow HTML caching.
 */
$conf['omit_vary_cookie'] = TRUE;
```

#### Varnish {#varnish}
Many high traffic sites will benefit from an extra layer of caching between the web server and the CDN. In these instances one or more Varnish reverse proxy servers is recommended.

#### HTTPS {#https}
SSL can be configured easily with [Let's Encrypt](https://letsencrypt.org/getting-started/). These certificates need to be renewed quarterly but this renewal process can be [automated](#automation).

#### HTTP/2 {#http2}
If you have configured HTTPS, you should go one step further and enable HTTP/2 to reap it's additional performance benefits. While HTTP/2 does not technically _require_ encryption, no browser currently supports it over HTTP, so for all intents and purposes HTTP/2 requires HTTPS. 

Enabling HTTP/2 is a [straight-forward process](https://blog.samat.org/2015/11/26/Enabling-HTTP2-on-Apache-2.4-on-Debian-Ubuntu/):

* Enable the Apache HTTP/2 mod:
```
sudo a2enmod http2
```
* Add the following line to the SSL vhost in question: 
```
Protocols h2 http/1.1
```
* Restart Apache
```
sudo service apache2 restart
```
