+++
title = 'Kube PI4'
date = 2026-06-03T00:06:08-07:00
draft = false
cover = "https://media1.tenor.com/m/80qlzG0ufboAAAAd/kubernetes.gif"
tags = ["ebpf", "k8s", "raspberrypi", "pi", "k3s", "kubernetes"]
keywords = ["bpf", "k8s", "raspberrypi", "k3s", "kubernetes"]
description = ""
showFullContent = false
readingTime = true
hideComments = false
toc = true
+++

## Background

This started as an “I’m bored” kind of project and ended up turning into something somewhat decent security-wise by getting information like alerts, security posture, and vulnerabilities in your Kubernetes cluster.

## Requirements.txt

* A Raspberry Pi 4 or better
* A decent-sized USB or drive you can connect over USB (I used 1TB, but you can get away with like 500GB)
* A static IP for your PI

## The Beginning

To start I went with an OS that’s not as minimal as something like Raspbian Lite and chose Ubuntu Server 2026 for as close to bleeding edge as possible and the least hacky as possible OS. The Downsides of it are that it’s a bit heavier on resources, but that’s ok since you’re not running any super-demanding charts like Prometheus.

1. Install K3S https://k3s.io/
2. Install Helm https://helm.sh/docs/intro/install/

Pretty easy so far yeah?
![alt text](/kubepi/hampter.jpeg)

## Installing Charts

Now that you installed the essentials, you’re going to install headlamp and add a plugin.

1. Follow the in-cluster installation for headlamp: https://headlamp.dev/#download-platforms

2. Add an ingress and middleware:

   ```yaml
   
   kind: Ingress
   apiVersion: networking.k8s.io/v1
   metadata:
    name: headlamp
    namespace: kube-system
    annotations:
     traefik.ingress.kubernetes.io/router.middlewares: kube-system-headlamp-auth@kubernetescrd
   
   spec:
    ingressClassName: traefik
    tls:
   
     secretName: headlamp
     hosts:
      - raspberrypi.local # change with your preferred hostname
     rules:
      host: raspberrypi.local # change with your preferred hostname
      http:
       paths:
        - path: /
          pathType: ImplementationSpecific
          backend:
           service:
            name: my-headlamp
            port:
             number: 80
   
   
   # for creating the tls secret do something like: `kubectl create secret tls my-app-certs --cert=path/to/domain.crt --key=path/to/domain.key --namespace=default` make sure your namespace is kube-system
   
   
   kind: Middleware
   apiVersion: traefik.io/v1alpha1
   metadata:
    name: headlamp-auth
    namespace: kube-system
    spec:
     basicAuth:
      secret: headlamp-auth-secret
   
   kind: Secret
   apiVersion: v1
   metadata:
    name: headlamp-auth-secret
    namespace: kube-system
   type: Opaque
   stringData:
   users: |
     pi:$2y$05$Q/XB.g6vYzkCy8iTXfISSenWaOtS5J4BDJMi01r7.CN6ZsoFUiL/C # raspberry as the password (can be changed referencing this: https://stackoverflow.com/questions/62116895/traefik-basic-auth )
   ```

3. Create a new cluster token for authenticating to headlamp `kubectl create token headlamp-admin -n kube-system`

4. Log in to the ui at https://raspberrypi.local or the hostname you put in the ingress

5. Now, you should be getting cluster metrics and data like running pods! To add the kubescape plugin, use this values.yaml file and upgrade your headlamp chart (you might need to issue a new cluster token and log back into your headlamp instance):

```yaml
config:
  pluginsDir: /build/plugins
initContainers:
  - command:
      - /bin/sh
      - '-c'
      - mkdir -p /build/plugins && cp -r /plugins/* /build/plugins/
    image: quay.io/kubescape/headlamp-plugin:v0.11.2
    name: kubescape-plugin
    volumeMounts:
      - mountPath: /build/plugins
        name: headlamp-plugins
```

6. Follow these docs for installing kubescape: https://kubescape.io/docs/install-operator/#prerequisites Now you have scanning in your cluster :smile:
7. If you wish to install the CLI too, follow the documentation here: https://kubescape.io/docs/install-cli

Nice, now you have a way to see your security posture, and kubescape gives recommendations on how to fix your posture too!

![image|226x223](/kubepi/hackerman.png)

This is where it’ll get hellish and a bit difficult but it won’t be too hard since I’ve already done it and can just give you the manifest files.

## This Is Where The Fun Begins!

![alt text](https://media4.giphy.com/media/v1.Y2lkPTc5MGI3NjExYXZlOGYzenYxdzM0YWg0b3Bqa2ZwZm81cjIydGl4Z2h0b2F0ejBtNiZlcD12MV9pbnRlcm5hbF9naWZfYnlfaWQmY3Q9Zw/NsIwMll0rhfgpdQlzn/giphy.gif)

1. Lets get kubearmor for runtime security now by following: https://docs.kubearmor.io/kubearmor/quick-links/deployment_guide No need to apply the test policy because we’re going to be bringing our own from: `https://github.com/kubearmor/policy-templates` I applied all of the CVE policies.
2. Install the CLI if you want: from: https://docs.kubearmor.io/kubearmor/quick-links/deployment_guide

Kubearmor done!!

Time to get Tetragon going too for security observability and more generic runtime enforcement

1. Follow the docs here for easy installation: https://tetragon.io/
2. Done! It’s that easy!

Now you have the most secure cluster in the world!! (Not really, but close enough on a trashy pi)

If you really wanna make it super l33t 1337 add alerting to discord!

1. Install fluent bit with this values.yaml file, make sure to install it into the logging namespace:

   ```yaml
   serviceAccount:
     create: true
     name: fluent-bit
   
   
   rbac:
     create: true
     nodeAccess: true
     eventsAccess: true
   
   clusterRole:
     create: true
     rules:
       - apiGroups: [""]
         resources: ["events", "pods", "namespaces", "nodes"]
         verbs: ["get", "list", "watch"]
   
   
   config:
     service: |
       [SERVICE]
           Flush        1
           Log_Level    info
           Parsers_File parsers.conf
           Parsers_File tetragon.conf
           HTTP_Server  On
           HTTP_Listen  0.0.0.0
           HTTP_Port    2020
   
     extraFiles: 
       tetragon.conf: |
         [PARSER]
             Name   tetragon_json
             Format json
             Time_Key    time
             Time_Format %Y-%m-%dT%H:%M:%S.%LZ
             Time_Keep   On
   
     inputs: |
       [INPUT]
           Name             tail
           Path             /var/log/kubearmor.log
           Tag              kubearmor
           Mem_Buf_Limit    10MB
           Skip_Long_Lines  On
   
       [INPUT]
           Name             tail
           Path             /var/run/cilium/tetragon/*.log
           Tag              tetragon
           Parser           tetragon_json
           Mem_Buf_Limit    10MB
           Skip_Long_Lines  On
           Refresh_Interval 5
   
       [INPUT]
           Name             tail
           Path             /var/log/pods/kubescape_kubescape-*/kubescape/*.log
           Tag              kubescape
           Mem_Buf_Limit    10MB
           Skip_Long_Lines  On
   
       [INPUT]
           Name             tail
           Path             /var/log/pods/kubescape_node-agent-*/node-agent/*.log
           Tag              kubescape.node-agent
           Mem_Buf_Limit    10MB
           Skip_Long_Lines  On
   
       [INPUT]
           Name             tail
           Path             /var/log/pods/kubescape_operator-*/operator/*.log
           Tag              kubescape.operator
           Mem_Buf_Limit    10MB
           Skip_Long_Lines  On
   
       [INPUT]
           Name             kubernetes_events
           Tag              k8s.events
   
     filters: |
       [FILTER]
           Name    grep
           Match   kubearmor
           Regex   log (DENY|BLOCK)
   
       [FILTER]
           Name    record_modifier
           Match   tetragon
           Record  has_tetragon_event true
   
   
       [FILTER]
           Name    grep
           Match   tetragon
           Exclude kubernetes.namespace_name ^kubearmor.*
       [FILTER]
           Name    grep
           Match   tetragon
           Exclude kubernetes.namespace ^kubearmor.*
       [FILTER]
           Name    grep
           Match   tetragon
           Exclude kubernetes.pod_name ^.*kubearmor.*$
       [FILTER]
           Name    grep
           Match   tetragon
           Exclude namespace kubearmor
   
       [FILTER]
           Name    grep
           Match   kubescape*
           Regex   log (HIGH|CRITICAL|FAILED|alert|unexpected|anomaly|malware)
   
       [FILTER]
           Name    grep
           Match   k8s.events
           Regex   log (Failed|Forbidden|Error|BackOff)
   
       [FILTER]
           Name               throttle
           Match              *
           Rate               5
           Window             30
           Print_Status       false
           Interval           1s
   
       [FILTER]
           Name    lua
           Match   *
           call    enrich
           code    function enrich(tag, timestamp, record) record["source"] = tag return 1, timestamp, record end
   
       [FILTER]
           Name    modify
           Match   *
           Add     cluster raspberrypi-security
   
     outputs: |
       [OUTPUT]
           Name             http
           Match            *
           Host             discord-proxy.logging.svc.cluster.local
           Port             8080
           URI              /
           Format           json_lines
           json_date_key    false
           tls              Off
           Retry_Limit      3
   
   daemonSetVolumes:
     - name: varlog
       hostPath:
         path: /var/log
     - name: tetragon
       hostPath:
         path: /var/run/cilium/tetragon
   
   daemonSetVolumeMounts:
     - name: varlog
       mountPath: /var/log
     - name: tetragon
       mountPath: /var/run/cilium/tetragon
   ```

2. You will get some crash loopbacks because the Discord proxy hasn’t been set up yet. Use this manifest to do it, remember to replace the env variable value for `DISCORD_WEBHOOK` with your webhook (yes, I did completely vibe code the proxy btw):

   ```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: discord-proxy
     namespace: logging
   data:
     server.js: |
       const http = require('http');
       const https = require('https');
       
       const WEBHOOK = process.env.DISCORD_WEBHOOK;
       
       // ------------------------------
       // Discord sender
       // ------------------------------
       function sendDiscord(username, content) {
         return new Promise((resolve, reject) => {
           if (content.length > 2000) content = content.slice(0, 1997) + '...';
       
           const body = JSON.stringify({
             username,
             avatar_url: 'https://i.kym-cdn.com/photos/images/newsfeed/003/132/154/b1b.jpg',
             content
           });
       
           const url = new URL(WEBHOOK);
       
           const req = https.request({
             hostname: url.hostname,
             path: url.pathname,
             method: 'POST',
             headers: {
               'Content-Type': 'application/json',
               'Content-Length': Buffer.byteLength(body)
             }
           }, res => {
             res.resume();
             resolve();
           });
       
           req.on('error', reject);
           req.write(body);
           req.end();
         });
       }
       
       // ------------------------------
       // Helpers
       // ------------------------------
       function safeJson(obj) {
         try {
           return JSON.stringify(obj, null, 2);
         } catch {
           return String(obj);
         }
       }
       
       function extractKubescapeAlert(record) {
         const f = record?.failure?.BaseRuntimeAlert;
         const proc = f?.identifiers?.process || {};
         const k8s = record?.failure?.RuntimeAlertK8sDetails || {};
       
         return {
           rule: record?.failure?.RuleAlert?.ruleDescription || 'Kubescape alert',
           alert: f?.alertName,
           severity: f?.severity,
           process: `${proc.name || 'unknown'} ${proc.commandLine || ''}`.trim(),
           namespace: k8s.namespace,
           pod: k8s.podName,
           node: k8s.nodeName,
           image: k8s.image
         };
       }
       
       function extractTetragon(record) {
         const p =
           record?.process_exec?.process ||
           record?.process_exit?.process;
       
         if (!p) return null;
       
         return {
           event: record.process_exec ? 'exec' : 'exit',
           binary: p.binary,
           args: p.arguments || '',
           pid: p.pid,
           pod: p.pod?.name,
           namespace: p.pod?.namespace || record.namespace
         };
       }
       
       // ------------------------------
       // Server
       // ------------------------------
       http.createServer((req, res) => {
         if (req.method !== 'POST') {
           res.writeHead(405);
           return res.end();
         }
       
         let body = '';
         req.on('data', d => body += d);
       
         req.on('end', async () => {
   
             let parsedBody;
   
             try {
               parsedBody = JSON.parse(body);
             } catch {
               parsedBody = null;
             }
           
             // ------------------------------
             // FAST PATH: direct forward based on username/content
             // ------------------------------
             if (
               parsedBody?.username === "Kubescape Scanner" &&
               typeof parsedBody?.content === "string"
             ) {
               await sendDiscord(parsedBody.username, parsedBody.content);
           
               res.writeHead(204);
               return res.end();
             }
   
   
   
           const lines = body.trim().split('\n');
       
           for (const line of lines) {
             if (!line) continue;
       
             try {
               const record = JSON.parse(line);
       
               const cluster = record.cluster || 'unknown';
               const source = record.source || record.tag || 'unknown';
               // Global namespace exclusion
               const namespace =
                 record?.failure?.RuntimeAlertK8sDetails?.namespace ||
                 record?.process_exec?.process?.pod?.namespace ||
                 record?.process_exit?.process?.pod?.namespace ||
                 record?.kubernetes?.namespace_name ||
                 record?.namespace;
               
               if (namespace === 'kubescape') {
                 continue;
               }
       
               // --------------------------
               // Kubescape alert handling
               // --------------------------
               if (record?.failure?.BaseRuntimeAlert) {
                 const a = extractKubescapeAlert(record);
       
                 const content =
       `🚨 **Kubescape Alert**
       Cluster: \`${cluster}\`
       Rule: ${a.rule}
       Alert: ${a.alert}
       Severity: ${a.severity}
       
       Process: \`${a.process}\`
       Pod: \`${a.pod}\`
       Namespace: \`${a.namespace}\`
       Node: \`${a.node}\``;
       
                 await sendDiscord('Kubescape', content);
                 continue;
               }
       
               // --------------------------
               // Tetragon handling (STRICT FILTER)
               // --------------------------
               if (
                 source === "tetragon" &&
                 (record.process_exec || record.process_exit)
               ) {
                 const ns =
                   record?.process_exec?.process?.pod?.namespace ||
                   record?.process_exit?.process?.pod?.namespace ||
                   record?.namespace;
       
                 // HARD GATE: only allowed namespaces
                 if (!ns || !["default", "kube-prod"].includes(ns)) {
                   continue;
                 }
       
                 const t = extractTetragon(record);
       
                 if (t) {
                   const content =
       `🧬 **Tetragon Event**
       Cluster: \`${cluster}\`
       Event: \`${t.event}\`
       
       Binary: \`${t.binary}\`
       Args: \`${t.args}\`
       Pod: \`${t.pod || 'host'}\`
       Namespace: \`${t.namespace || 'host'}\``;
       
                   await sendDiscord('Tetragon', content);
                   continue;
                 }
               }
       
               // --------------------------
               // Fallback
               // --------------------------
               const pretty = safeJson(record);
       
               const content =
       `**[${cluster}] ${source}**
       \`\`\`json
       ${pretty}
       \`\`\``;
       
               await sendDiscord('Security Logs', content);
       
             } catch (e) {
               console.error('parse error', e, line);
             }
           }
       
           res.writeHead(204);
           res.end();
         });
       
       }).listen(8080, () => console.log('discord proxy listening on 8080'));
   ---
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: discord-proxy
     namespace: logging
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: discord-proxy
     template:
       metadata:
         labels:
           app: discord-proxy
       spec:
         containers:
           - name: proxy
             image: node:20-alpine
             command: ["node", "/app/server.js"]
             env:
               - name: DISCORD_WEBHOOK
                 value: "https://discord.com/api/webhooks/yourwebhookhere!"
             volumeMounts:
               - name: proxy-script
                 mountPath: /app
         volumes:
           - name: proxy-script
             configMap:
               name: discord-proxy
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: discord-proxy
     namespace: logging
   spec:
     selector:
       app: discord-proxy
     ports:
       - port: 8080
         targetPort: 8080
   ```

3. For the kubescape reporter, it’ll run a basic scan for compliance and CVEs; it will not scan your images for vulns without it being specified (this was also vibe coded), scans will run at 8am every day automatically:

   ```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: kubescape-reporter-script
     namespace: kubescape
   data:
     scan-and-send.js: |
       const { execSync } = require('child_process');
       const http = require('http');
       const fs = require('fs');
       const zlib = require('node:zlib');
       const { promisify } = require('node:util');
       
       const brotliCompress = promisify(zlib.brotliCompress);
       
       async function shrinkString(hugeString) {
         const inputBuffer = Buffer.from(hugeString, 'utf8');
       
         const compressedBuffer = await brotliCompress(inputBuffer, {
           params: {
             [zlib.constants.BROTLI_PARAM_MODE]:
               zlib.constants.BROTLI_MODE_TEXT,
             [zlib.constants.BROTLI_PARAM_QUALITY]: 11,
             [zlib.constants.BROTLI_PARAM_LGWIN]: 24
           }
         });
       
         return compressedBuffer.toString('base64url');
       }
       
       function sendToProxy(username, content) {
         return new Promise((resolve, reject) => {
           const payload = JSON.stringify({
             username,
             content
           });
       
           const req = http.request(
             {
               hostname: 'discord-proxy.logging.svc.cluster.local',
               port: 8080,
               path: '/',
               method: 'POST',
               headers: {
                 'Content-Type': 'application/json',
                 'Content-Length': Buffer.byteLength(payload)
               }
             },
             res => {
               res.resume();
               resolve();
             }
           );
       
           req.on('error', reject);
           req.write(payload);
           req.end();
         });
       }
       
       async function main() {
         try {
           console.log('📥 Installing Kubescape...');
       
           execSync(
             'curl -s https://raw.githubusercontent.com/kubescape/kubescape/master/install.sh | /bin/bash',
             { stdio: 'inherit' }
           );
       
           const kubescapePath = '/usr/local/bin/kubescape';
       
           if (!fs.existsSync(kubescapePath)) {
             throw new Error(
               'Kubescape binary not found after install.'
             );
           }
       
           console.log('🔄 Running Kubescape scan...');
       
           try {
             execSync(
               `${kubescapePath} scan --format json --output /tmp/results.json`,
               {
                 stdio: 'inherit',
                 timeout: 300_000
               }
             );
           } catch (err) {
             console.warn(
               '⚠️ Full scan failed, retrying with --include-namespaces all...'
             );
       
             execSync(
               `${kubescapePath} scan --include-namespaces all --format json --output /tmp/results.json`,
               {
                 stdio: 'inherit',
                 timeout: 300_000
               }
             );
           }
       
           if (!fs.existsSync('/tmp/results.json')) {
             throw new Error('Scan output file not found.');
           }
       
           const rawData = fs.readFileSync(
             '/tmp/results.json',
             'utf8'
           );
       
           const report = JSON.parse(rawData);
       
           const vulnerabilities = report.summaryDetails?.vulnerabilities || {};
   
           const cves = vulnerabilities.CVEs || [];
   
           const cveSeveritySummary = vulnerabilities.mapsSeverityToSummary || {};
           const criticalCVEs = cveSeveritySummary.Critical || 0;
   
           const highCVEs = cveSeveritySummary.High || 0;
   
           const mediumCVEs = cveSeveritySummary.Medium || 0;
   
           const lowCVEs = cveSeveritySummary.Low || 0;
           const topCVEs = cves.sort((a, b) => {
             const sev = {
               Critical: 4,
               High: 3,
               Medium: 2,
               Low: 1
               };
             return (
             (sev[b.severity] || 0) -
             (sev[a.severity] || 0)
             );
           }).slice(0, 10);
   
           const summary =
             report.summaryDetails || {};
       
           const controls =
             summary.controls || {};
       
           const frameworks =
             summary.frameworks || [];
       
           const severityCounters =
             summary.controlsSeverityCounters || {};
       
           const critical =
             severityCounters.criticalSeverity || 0;
       
           const high =
             severityCounters.highSeverity || 0;
       
           const medium =
             severityCounters.mediumSeverity || 0;
       
           const low =
             severityCounters.lowSeverity || 0;
       
           const failedFrameworks = frameworks
             .filter(f => f.status === 'failed')
             .map(f => ({
               name: f.name,
               score: f.complianceScore,
               failed:
                 f.ResourceCounters?.failedResources || 0
             }));
       
           const failedControls = Object.values(
             controls
           )
             .filter(c => c.status === 'failed')
             .map(c => ({
               id: c.controlID,
               name: c.name,
               severity: c.severity,
               failedResources:
                 c.ResourceCounters?.failedResources || 0,
               complianceScore: c.complianceScore
             }));
       
           const severityBreakdown = {
             Critical: 0,
             High: 0,
             Medium: 0,
             Low: 0
           };
       
           for (const control of failedControls) {
             if (
               Object.prototype.hasOwnProperty.call(
                 severityBreakdown,
                 control.severity
               )
             ) {
               severityBreakdown[control.severity]++;
             }
           }
       
           const topControls = [...failedControls]
             .sort(
               (a, b) =>
                 b.failedResources - a.failedResources
             )
             .slice(0, 10);
       
           const compressedReport =
             await shrinkString(rawData);
       
           let emoji = '🟢';
       
           if (critical > 0) {
             emoji = '🔴';
           } else if (high > 0) {
             emoji = '🟠';
           } else if (medium > 0) {
             emoji = '🟡';
           }
       
           const frameworkSummary =
             failedFrameworks.length > 0
               ? failedFrameworks
                   .map(
                     f =>
                       `• ${f.name}: ${f.failed} failed resources (${f.score.toFixed(
                         1
                       )}% compliant)`
                   )
                   .join('\n')
               : 'None';
       
           const findingsSummary =
             topControls.length > 0
               ? topControls
                   .map(
                     c =>
                       `• [${c.severity}] ${c.name} (${c.failedResources} resources)`
                   )
                   .join('\n')
               : 'None';
       
           const message = [
             `${emoji} **Kubescape Security Report**`,
             `Scan Date: ${new Date().toISOString()}`,
             '',
             '**Framework Failures**',
             frameworkSummary,
             '',
             '**Control Findings**',
             `• Critical: ${severityBreakdown.Critical}`,
             `• High: ${severityBreakdown.High}`,
             `• Medium: ${severityBreakdown.Medium}`,
             `• Low: ${severityBreakdown.Low}`,
             '',
             '**Top Failed Controls**',
             findingsSummary,
             '',
             '**CVE Findings**',
             `• Critical: ${criticalCVEs}`,
             `• High: ${highCVEs}`,
             `• Medium: ${mediumCVEs}`,
             `• Low: ${lowCVEs}`,
           ].join('\n');
           if (topCVEs.length > 0) {
               messageParts.push(
                 '',
                 '**Top CVEs**',
                 ...topCVEs.map(
                   cve =>
                     `• ${cve.name || cve.cveID || cve.id} [${cve.severity}]`
                 )
               );
           }
       
           console.log(message);
       
           console.log(
             '📤 Sending summary to Discord proxy...'
           );
       
           await sendToProxy(
             'Kubescape Scanner',
             message
           );
       
      
           console.log('✅ Done.');
         } catch (err) {
           console.error('❌ Fatal error:', err);
           process.exit(1);
         }
       }
       
       main();
   
   
   ---
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: kubescape-reporter
     namespace: kubescape
   ---
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRoleBinding
   metadata:
     name: kubescape-reporter-admin-binding
   roleRef:
     apiGroup: rbac.authorization.k8s.io
     kind: ClusterRole
     name: cluster-admin
   subjects:
     - kind: ServiceAccount
       name: kubescape-reporter
       namespace: kubescape
   ---
   apiVersion: batch/v1
   kind: CronJob
   metadata:
     name: kubescape-discord-reporter
     namespace: kubescape
   spec:
     schedule: "0 8 * * *"
     jobTemplate:
       spec:
         template:
           spec:
             serviceAccountName: kubescape-reporter
             restartPolicy: Never
             containers:
               - name: real-scanner
                 # Using the full-fat bookworm image to ensure curl and bash are pre-installed
                 image: node:20-bookworm
                 command: ["node", "/scripts/scan-and-send.js"]
                 volumeMounts:
                   - name: script
                     mountPath: /scripts
             volumes:
               - name: script
                 configMap:
                   name: kubescape-reporter-script
   ```
alerts should look like this:
![alt text](/kubepi/alert_example1.png)

and
![alt text](/kubepi/alert_example2.png)

Have fun!
