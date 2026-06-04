+++
title = 'Kube PI4'
date = "2026-06-03T00:06:08-07:00"
draft = false
coverOnly = true
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

2. Add an ingress, middleware, and service account:

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
   
   ---
   kind: Middleware
   apiVersion: traefik.io/v1alpha1
   metadata:
    name: headlamp-auth
    namespace: kube-system
    spec:
     basicAuth:
      secret: headlamp-auth-secret
   ---
   kind: Secret
   apiVersion: v1
   metadata:
    name: headlamp-auth-secret
    namespace: kube-system
   type: Opaque
   stringData:
   users: |
     pi:$2y$05$Q/XB.g6vYzkCy8iTXfISSenWaOtS5J4BDJMi01r7.CN6ZsoFUiL/C # raspberry as the password (can be changed referencing this: https://stackoverflow.com/questions/62116895/traefik-basic-auth )
   ---
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: headlamp-admin
     namespace: kube-system

   ---
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRoleBinding
   metadata:
     name: headlamp-admin
   roleRef:
     apiGroup: rbac.authorization.k8s.io
     kind: ClusterRole
     name: cluster-admin
   subjects:
     - kind: ServiceAccount
       name: headlamp-admin
       namespace: kube-system
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
   
               if (namespace === 'kubescape' || namespace === 'kubearmor') {
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
               if (source === 'kubescape.node-agent'){
                 continue;
               }
   
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

    const EXCLUDED_NAMESPACES = ['kubescape', 'logging', 'kube-system', 'kubearmor'];

    const brotliCompress = promisify(zlib.brotliCompress);

    async function shrinkString(hugeString) {
      const inputBuffer = Buffer.from(hugeString, 'utf8');
      const compressedBuffer = await brotliCompress(inputBuffer, {
        params: {
          [zlib.constants.BROTLI_PARAM_MODE]: zlib.constants.BROTLI_MODE_TEXT,
          [zlib.constants.BROTLI_PARAM_QUALITY]: 11,
          [zlib.constants.BROTLI_PARAM_LGWIN]: 24
        }
      });
      return compressedBuffer.toString('base64url');
    }

    function sendToProxy(username, content) {
      return new Promise((resolve, reject) => {
        const payload = JSON.stringify({ username, content });
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
          res => { res.resume(); resolve(); }
        );
        req.on('error', reject);
        req.write(payload);
        req.end();
      });
    }

    function getClusterImages() {
      try {
        const tmplFile = '/tmp/kubectl-tmpl.txt';
        fs.writeFileSync(tmplFile,
          '{{range .items}}' +
          '{{$ns := .metadata.namespace}}' +
          '{{range .spec.containers}}{{$ns}}\t{{.image}}\n{{end}}' +
          '{{range .spec.initContainers}}{{$ns}}\t{{.image}}\n{{end}}' +
          '{{end}}'
        );

        const out = execSync(
          `kubectl get pods --all-namespaces -o go-template-file=${tmplFile}`,
          { encoding: 'utf8', timeout: 30_000 }
        );

        const excluded = new Set(EXCLUDED_NAMESPACES);
        const images = new Set();

        for (const line of out.split('\n')) {
          const tab = line.indexOf('\t');
          if (tab === -1) continue;
          const namespace = line.slice(0, tab).trim();
          const image     = line.slice(tab + 1).trim();
          if (!image || excluded.has(namespace)) continue;
          images.add(image);
        }

        console.log(`🔍 Discovered ${images.size} unique images (excluding system namespaces)`);
        return [...images];
      } catch (err) {
        console.warn('⚠️ Image discovery failed:', err.message.split('\n')[0]);
        return [];
      }
    }

    function scanImage(kubescapePath, image) {
      const outFile = `/tmp/img-${Date.now()}.json`;
      try {
        execSync(
          `${kubescapePath} scan image "${image}" --format json --output ${outFile} 2>/dev/null`,
          { stdio: 'pipe', timeout: 120_000 }
        );
      } catch (_) {
        // non-zero exit is normal when vulnerabilities are found
      }
      try {
        if (!fs.existsSync(outFile)) return null;
        return JSON.parse(fs.readFileSync(outFile, 'utf8'));
      } catch (_) {
        return null;
      } finally {
        if (fs.existsSync(outFile)) fs.unlinkSync(outFile);
      }
    }

    function parseImageVulns(report) {
      if (!report) return null;
      const matches =
        report?.spec?.payload?.matches ||
        report?.matches ||
        [];
      if (!matches.length) return null;

      const counts = { Critical: 0, High: 0, Medium: 0, Low: 0 };
      const cves = [];
      for (const m of matches) {
        const sev = m?.vulnerability?.severity || 'Unknown';
        const id  = m?.vulnerability?.id       || '?';
        const pkg = m?.artifact?.name          || '';
        const ver = m?.artifact?.version       || '';
        if (sev in counts) counts[sev]++;
        cves.push({ id, severity: sev, pkg, ver });
      }

      const sevRank = { Critical: 4, High: 3, Medium: 2, Low: 1 };
      const topCVEs = [...cves]
        .sort((a, b) => (sevRank[b.severity] || 0) - (sevRank[a.severity] || 0))
        .slice(0, 3);

      return { ...counts, topCVEs };
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
          throw new Error('Kubescape binary not found after install.');
        }

        console.log('📥 Installing kubectl...');
        execSync(
          'KUBECTL_VERSION=$(curl -sL https://dl.k8s.io/release/stable.txt) && ' +
          'curl -sLO "https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/arm64/kubectl" && ' +
          'chmod +x kubectl && mv kubectl /usr/local/bin/kubectl && kubescape download artifacts --output ~/kubescape-offline-data/',
          { stdio: 'inherit', shell: '/bin/bash' }
        );

        // ── Pass 1: compliance scan ───────────────────────────────────────
        console.log('🔄 Running compliance scan...');
        //        execSync(
        //          `${kubescapePath} scan framework AllControls,ArmoBest,DevOpsBest,MITRE,NSA,SOC2,cis-v1.10.0,cis-v1.12.0 --include-namespaces all --format json --output /tmp/results.json`,
        //          { stdio: 'inherit', timeout: 300_000 }
        //        );

        execSync(
          `${kubescapePath} scan framework AllControls,ArmoBest,DevOpsBest,MITRE,NSA,SOC2,cis-v1.10.0,cis-v1.12.0 --include-namespaces all --format json --use-artifacts-from ~/kubescape-offline-data/ --output /tmp/results.json`,
          { stdio: 'inherit', timeout: 300_000 }
        );
        if (!fs.existsSync('/tmp/results.json')) {
          throw new Error('Compliance scan output not found.');
        }

        const rawData = fs.readFileSync('/tmp/results.json', 'utf8');
        const report  = JSON.parse(rawData);

        // ── Pass 2: per-image vulnerability scans ────────────────────────
        const allImages    = getClusterImages();
        const MAX_IMAGES   = 20;
        const imagesToScan = allImages.slice(0, MAX_IMAGES);

        if (allImages.length > MAX_IMAGES) {
          console.warn(`⚠️ ${allImages.length} images found — scanning first ${MAX_IMAGES}`);
        }

        const imageResults = {};
        //for (const image of imagesToScan) {
        //  console.log(`  🔬 Scanning ${image}`);
        //  const parsed = parseImageVulns(scanImage(kubescapePath, image));
        //  if (parsed) imageResults[image] = parsed;
        //}

        // Aggregate CVE totals across all scanned images
        let criticalCVEs = 0, highCVEs = 0, mediumCVEs = 0, lowCVEs = 0;
        for (const d of Object.values(imageResults)) {
          criticalCVEs += d.Critical;
          highCVEs     += d.High;
          mediumCVEs   += d.Medium;
          lowCVEs      += d.Low;
        }

        // ── Controls / frameworks ─────────────────────────────────────────
        const summary          = report.summaryDetails || {};
        const controls         = summary.controls      || {};
        const frameworks       = summary.frameworks    || [];
        const severityCounters = summary.controlsSeverityCounters || {};
        const critical = severityCounters.criticalSeverity || 0;
        const high     = severityCounters.highSeverity     || 0;
        const medium   = severityCounters.mediumSeverity   || 0;

        //const failedFrameworks = frameworks
        //  .filter(f => f.status === 'failed')
        const failedFrameworks = frameworks.map(f => ({
            name:   f.name,
            score:  f.complianceScore,
            failed: f.ResourceCounters?.failedResources || 0
          }));

        const failedControls = Object.values(controls)
          .filter(c => c.status === 'failed')
          .map(c => ({
            id:              c.controlID,
            name:            c.name,
            severity:        c.severity,
            failedResources: c.ResourceCounters?.failedResources || 0,
            complianceScore: c.complianceScore
          }));

        const severityBreakdown = { Critical: 0, High: 0, Medium: 0, Low: 0 };
        for (const control of failedControls) {
          if (control.severity in severityBreakdown) {
            severityBreakdown[control.severity]++;
          }
        }

        const topControls = [...failedControls]
          .sort((a, b) => b.failedResources - a.failedResources)
          .slice(0, 10);

        const compressedReport = await shrinkString(rawData);

        // ── Emoji ─────────────────────────────────────────────────────────
        let emoji = '🟢';
        if (critical > 0 || criticalCVEs > 0)  emoji = '🔴';
        else if (high > 0 || highCVEs > 0)      emoji = '🟠';
        else if (medium > 0 || mediumCVEs > 0)  emoji = '🟡';

        // ── Message ───────────────────────────────────────────────────────
        const frameworkSummary = failedFrameworks.length > 0
          ? failedFrameworks.map(f =>
              `• ${f.name}: ${f.failed} failed resources (${f.score.toFixed(1)}% compliant)`
            ).join('\n')
          : 'None';

        const findingsSummary = topControls.length > 0
          ? topControls.map(c =>
              `• [${c.severity}] ${c.name} (${c.failedResources} resources)`
            ).join('\n')
          : 'None';

        const sortedImages = Object.entries(imageResults)
          .sort(([, a], [, b]) => {
            if (b.Critical !== a.Critical) return b.Critical - a.Critical;
            return b.High - a.High;
          });

        const imageSection = sortedImages.length > 0
          ? sortedImages.slice(0, 5).map(([img, data]) => {
              const lines = [
                `• \`${img}\`: C:${data.Critical} H:${data.High} M:${data.Medium} L:${data.Low}`
              ];
              for (const cve of data.topCVEs) {
                lines.push(`  ↳ ${cve.id} [${cve.severity}] ${cve.pkg}${cve.ver ? '@' + cve.ver : ''}`);
              }
              return lines.join('\n');
            }).join('\n')
          : 'None (no images found or all scans failed)';
        const overallScoreCompliance =
          report?.summaryDetails?.complianceScore ??
          report?.summaryDetails?.resourcesScore ??
          frameworks?.reduce((a, f) => a + (f.complianceScore || 0), 0) /
            (frameworks?.length || 1);
        const messageParts = [
          `${emoji} **Kubescape Security Report**`,
          `Scan Date: ${new Date().toISOString()}`,
          `Overall: ${overallScoreCompliance.toFixed(2) ?? 'N/A'}%`,
          '',
          '**Framework Results**',
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

        ];

        if (sortedImages.length > 5) {
          messageParts.push(`_(${sortedImages.length - 5} more images with findings omitted)_`);
        }

        const message = messageParts.join('\n');

        console.log(message);
        console.log('📤 Sending summary to Discord proxy...');
        await sendToProxy('Kubescape Scanner', message);
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
