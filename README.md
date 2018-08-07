## Presentation - Things to check before you deploy NodeJS apps to prod

## Using up to date base images
- Many issues get patched every now and then, in particular [CVEs](https://nodejs.org/en/blog/vulnerability/june-2018-security-releases/)
- Issues can be quite scary, and there is no excuse for ignorance
- We periodically repackage base NodeJS images in our private ECR repo
- I have written a script to automatically detect when these are out of date to make the process much simpler and easier

```bash
#!/usr/local/bin/bash

ecr_location='278521702583.dkr.ecr.us-west-2.amazonaws.com'

declare -A MAP

for i in $(egrep -oR "${ecr_location}.*$" * | grep -v 'node_modules'); do
  IFS=':' read -r -a triple <<< "${i}"

  if [ "${#triple[@]}" -ne 3 ]; then
    continue;
  fi

  file=${triple[0]}
  tag=${triple[2]}
  repo_name=$(echo ${triple[1]} | sed -e "s/${ecr_location}\///g")

  if [ -z ${MAP[${repo_name}]+_} ]; then
    tag=$(aws ecr describe-images  --registry-id=278521702583 --repository-name=$repo_name | jq .imageDetails[].imageTags[] | sort | tail -1 | sed -e 's/"//g')

    MAP[$repo_name]=$tag
  fi

  latest="${MAP[$repo_name]}"

  if [ "${tag}" \< "${latest}" ]; then
    echo "${file} ${repo_name} ${tag} -> ${latest}"
  fi
done

```

Sample output

```
daryl@daryl-Inspiron-7537 [14:32:36] [~/projects/pmsx] [test_bk_block_step]
-> % check_for_outdates_images .  

components/domain-builder/docker-compose.yaml arch/node-v8-kcl-190-builder v1.0.2 -> v1.1.0
components/domain-builder/Dockerfile arch/node-v8-kcl-190 v1.0.6 -> v1.1.0
components/ari-stream/docker-compose.yaml arch/node-v8-kcl-190-builder v1.0.2 -> v1.1.0
components/ari-stream/Dockerfile arch/node-v8-kcl-190 v1.0.6 -> v1.1.0

```

## Security patching your NodeJS apps

- NodeJS echo system has millions of packages
- You can indirectly pull in an insecure package because its a dependency of your dependency
- This sounds like a lot of work
- The cool thing is there are projects that are out there that can help you with this

I recommend using `npm audit`, it will look at your package lock and compare against its online database
of know vulnerabilities. It will even provide options to fix issues (```--fix```). There will be the odd case where you may
have to do a fix yourself.

One annoying aspect of npm audit is that it will also scan your dev dependencies, hence will return a non zero exit code in this case.
This feature is only available in npm 6.1.0


```
daryl@daryl-Inspiron-7537 [14:57:53] [~/projects/bematech/components/api] [develop *]
-> % npm audit            
                                                                                
                       === npm audit security report ===                        
                                                                                
┌──────────────────────────────────────────────────────────────────────────────┐
│                                Manual Review                                 │
│            Some vulnerabilities require your attention to resolve            │
│                                                                              │
│         Visit https://go.npm.me/audit-guide for additional guidance          │
└──────────────────────────────────────────────────────────────────────────────┘
┌───────────────┬──────────────────────────────────────────────────────────────┐
│ Moderate      │ Memory Exposure                                              │/
├───────────────┼──────────────────────────────────────────────────────────────┤
│ Package       │ bl                                                           │
├───────────────┼──────────────────────────────────────────────────────────────┤
│ Patched in    │ >=0.9.5 <1.0.0 || >=1.0.1                                    │
├───────────────┼──────────────────────────────────────────────────────────────┤
│ Dependency of │ pryjs [dev]                                                  │
├───────────────┼──────────────────────────────────────────────────────────────┤
│ Path          │ pryjs > pygmentize-bundled > bl                              │
├───────────────┼──────────────────────────────────────────────────────────────┤
│ More info     │ https://nodesecurity.io/advisories/596                       │
└───────────────┴──────────────────────────────────────────────────────────────┘
found 1 moderate severity vulnerability in 2451 scanned packages
  1 vulnerability requires manual review. See the full report for details.


```

## Setting environmental variable NODE_ENV to 'production'

A lot of applications, in particular [ExpressJS](https://expressjs.com/) have switches. In the case of Express, by default it
will render stack traces and debug flags. This can slow down your app down significantly - see this [article](https://www.dynatrace.com/news/blog/the-drastic-effects-of-omitting-node_env-in-your-express-js-applications/)
for more details


## Using the Trust Proxy settings for Express JS

By default ExpressJS will not honour the X-Forwarded-* header, for us this means that
if you deploy an app in Kubernetes the remote address logged will be the ip address of the
ELB - this is not very ideal - particular if its for audit logging

You can turn this setting on for your express app to have the right ip address showing up

```javascript

app.set('trust proxy', true)

```

As always read the docs if you require [custom](https://expressjs.com/en/guide/behind-proxies.html) configuration.

Now if you do log the ip address as a field, it will show up correctly and further more you can render a world map of where traffic is coming from in graylog

![Graylog world map](http://docs.graylog.org/en/2.4/_images/geolocation_8.png)


## Using reader endpoints of Aurora + SSL

Most of the code I have seen only makes use of the read write Aurora master.
This is a waste of resources in my opinion because we should be routing read only queries
to the read only replicas. This way we can spread the load. The mysql2 driver has the Amazon RDS certificates packaged with it, so we should be connection over an encrypted
connection because we should abide by the principle of security in layers

Here is the sample Sequelize configuration

```javascript

const dbconfig = {
  dialect: 'mysql',
  logging: config('SEQUELIZE_LOG') ? console.log : false,
  replication: {
    read: [
      { host: 'test-ro-xxxxxxxxxxx.us-west-2.rds.amazonaws.com', username: 'app', password: '******' }
    ],
    write: { host: 'test-xxxxxxxxxxx.us-west-2.rds.amazonaws.com', username: 'app', password: '******' }
  },
  pool: {
    max: config('MYSQL_MAX_CONNECTIONS'),
    min: 1,
    idle: 10000
  },
  operatorsAliases: false,
  dialectOptions : {
    ssl: "Amazon RDS"
  }
}

const sequelize = new Sequelize(database, username, password, dbconfig)

```

There is a [PR](https://github.com/siteminder-au/arch-packages/pull/7/files) being worked on by Nathan Pickworth in arch-packages which will deal with most / all of the concerns


## Metrics

Everyone one knows the metrics are important, but you have to be extremely careful what you measure and how you measure it.
For instance if one wants to see the response time per route you could do something like this.

```javascript

const labels = { method: req.method, path: req.path }

metrics.histogram('http_request_count', 1, labels)

```

Seems pretty harmless but it isn't, a new metric is created in prometheus for each unique combination of labels.
Now if your request contains path parameters, you can quite quickly end up with a potentially unbounded number of metrics.

```javascript

bematech:api:http_request_count {
  app="bematech-api",
  instance="100.98.220.32:3000",
  path="/reservations",
  pod_template_hash="892397533"
}

bematech:api:http_request_count {
  app="bematech-api",
  instance="100.98.220.32:3000",
  path="/reservations/TRS-45431-1/events/2018-04-06T15:07:18+00:00/document"
}

bematech:api:http_request_count {
  app="bematech-api",
  instance="100.98.220.32:3000",
  path="/reservations/EXP-32484-1/events/2018-04-06T15:12:51+00:00/document"
}

bematech:api:http_request_count {
  app="bematech-api",
  instance="100.98.220.32:3000",
  path="/reservations/BDC-86757-2/events/2018-04-06T15:02:18+00:00/document"
}

 bematech:api:http_request_count {
   app="bematech-api",
   instance="100.98.220.32:3000",
   path="/reservations/BDC-86757-1/events/2018-04-06T15:02:18+00:00/document"
 }

```
