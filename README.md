# DockerLabs Penetration Testing Portfolio

<div align="center">

![Platform](https://img.shields.io/badge/Platform-DockerLabs-critical?style=flat-square)
![Machines](https://img.shields.io/badge/Machines-74-blue?style=flat-square)
![Difficulty](https://img.shields.io/badge/Difficulty-Easy-informational?style=flat-square)
![Status](https://img.shields.io/badge/Status-Active-success?style=flat-square)
![Language](https://img.shields.io/badge/Language-English-lightgrey?style=flat-square)

**Professional Penetration Testing Documentation and Technical Lab Reports**

</div>

---

## Overview

This repository contains comprehensive technical documentation for penetration testing exercises conducted on the DockerLabs platform. DockerLabs is a community driven, container based boot2root platform where each machine is shipped as a Docker archive deployed locally through an automated provisioning script. The collection represents systematic security analysis across 74 machines, emphasizing methodological rigor and professional reporting standards over simple flag acquisition.

## Portfolio Philosophy

**Quality Over Quantity**

This repository differs from traditional Capture The Flag writeups. Each document serves as a technical lab report that demonstrates:

- Systematic vulnerability analysis and exploitation methodology
- Complete attack chain documentation from reconnaissance to privilege escalation
- Reproducible technical evidence through synchronized terminal logs and visual proof
- Professional reporting standards suitable for security assessment documentation

The objective is to showcase not merely the ability to compromise systems, but the capacity to document security findings with the precision and clarity expected in professional penetration testing engagements.

**Target Audience**

These reports are designed for security engineers, penetration testers, red team operators, and technical hiring managers evaluating practical security analysis capabilities and technical documentation proficiency.

---

## Repository Structure

```
dockerlabs-writeups/
│
├── <machinename>/                # Per machine penetration test
│   ├── <machinename>.md          # Complete technical writeup
│   └── image*.png                # Visual evidence and screenshots (loose in dir)
│
└── README.md
```

Each machine lives in its own directory, named identically to the machine itself, and the writeup file carries the same name as the directory. Screenshots are stored loose inside the machine directory (`image.png`, `image-1.png`, `image-2.png`, and so on) and embedded inline at the relevant technical step.

**Deployment Convention**

DockerLabs machines are provisioned locally with the platform provided `auto_deploy.sh` script, which loads the `.tar` archive and exposes the container on the Docker bridge network, typically assigning an address in the `172.17.0.0/16` range. Each engagement begins by deploying the container and recording the assigned IP before scanning.

**Repository Statistics:**

- 74 completed machine writeups with full technical documentation
- Consistent executive summary format implemented across all reports
- Visual evidence integrated inline throughout all reports
- Predominantly Easy and Very Easy difficulty, covering a wide spread of web and service exploitation techniques

---

## Documentation Workflow

The repository employs a structured pipeline that ensures consistency and technical accuracy across every report.

**Phase 1: Manual Engagement and Data Collection**

Each DockerLabs machine is solved manually, without AI assistance, as an authentic penetration test exercise. During the engagement:

- The Docker archive is deployed locally and the assigned container IP is recorded
- Terminal session logs and observations are captured
- Visual evidence is collected through screenshots placed in the machine directory
- All artifacts are preserved with their original filenames

**Phase 2: Report Generation**

The raw material is reorganized into the standardized reporting format:

- Terminal logs are structured into the canonical section layout
- Visual evidence is embedded inline at the relevant technical steps
- Commands and outputs are preserved verbatim for reproducibility
- The attack chain is synthesized from the exploitation sequence

**Phase 3: Quality Assurance**

- The executive summary is generated as a dense technical narrative from the findings
- The attack chain is validated against the actual exploitation path
- Visual proof is verified for accuracy and completeness
- The final report is formatted according to the standardized template

This workflow ensures that each of the 74 machine reports maintains consistent quality and professional presentation standards while preserving the integrity of the original manual engagement.

---

## Standardized Reporting Format

Every writeup in this repository adheres to a consistent technical reporting structure designed to ensure clarity, reproducibility, and professional presentation.

### Report Structure Template

```markdown
# [Machine Name]

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| [Name] | [Author] | [Difficulty] | dockerlabs |

**Summary:** [Comprehensive overview of vulnerabilities, attack vectors, and exploitation path]

---

## Reconnaissance
[Network discovery, port scanning, service enumeration, and initial information gathering]

## Initial Access
[Vulnerability identification, exploitation techniques, and initial foothold acquisition]

## Lateral Movement
[Optional: user to user transitions before root escalation]

## Privilege Escalation
[Enumeration of privilege escalation vectors and root access methodology]

---

## Attack Chain Summary
1. **Reconnaissance**: [Specific enumeration findings]
2. **Vulnerability Discovery**: [Identified attack surface]
3. **Exploitation**: [Exploitation method and user access]
4. **Internal Enumeration**: [Post-exploitation information gathering]
5. **Privilege Escalation**: [Root access technique]
```

### Reporting Conventions

- **Verbatim Output**: All tool output (nmap, gobuster, ffuf, sqlmap, hydra, john, wpscan) is reproduced in full inside fenced code blocks, never truncated or redacted
- **Shell Integrity**: The original terminal prompt banner and reverse shell upgrade sequences are preserved exactly as recorded
- **Visual Proof**: Screenshots are embedded immediately after the technical step they illustrate, using their original filenames
- **Narrative Prose**: Fluid paragraphs with continuous numbered steps, avoiding hyphenated bullet aesthetics
- **Dense Summary**: A single paragraph executive summary telling the exploitation chain as a continuous story

---

## Technical Skills Matrix

The writeups in this repository demonstrate proficiency across the complete penetration testing lifecycle.

### Reconnaissance and Enumeration

| Skill Area | Tools and Techniques |
|------------|---------------------|
| Network Scanning | Nmap full port scans, service and script detection, UDP enumeration |
| Service Fingerprinting | Version detection, banner grabbing, vulnerability correlation |
| Web Enumeration | Gobuster, feroxbuster, ffuf, directory and parameter brute forcing |
| Virtual Host Discovery | Host header fuzzing, subdomain enumeration, vhost routing |
| User Enumeration | SMB anonymous shares, FTP, WordPress author enumeration, OSINT |

### Vulnerability Analysis

| Skill Area | Tools and Techniques |
|------------|---------------------|
| Web Application Testing | SQL injection, parameter fuzzing, manual source code analysis |
| Local File Inclusion | Path traversal, PHP filter chains, AJP Ghostcat, log poisoning |
| File Upload Vulnerabilities | Extension and content type bypass, web shell deployment |
| Configuration Analysis | Default credentials, exposed .env and config files, misconfigured services |
| Steganography | EXIF metadata analysis, image comparison, hidden credential recovery |

### Exploitation

| Skill Area | Tools and Techniques |
|------------|---------------------|
| Remote Code Execution | Web shell deployment, reverse shell establishment, command injection |
| Service Exploitation | Jenkins, Tomcat, Grafana, WordPress, Drupal, Joomla, Nibbleblog |
| Authentication Bypass | SQLi auth bypass, JWT cracking, Hydra credential brute forcing |
| Known CVE Exploitation | vsftpd 2.3.4 backdoor, OpenSSH user enum, Drupalgeddon 2, Grafana dir traversal, Jenkins CVE-2024-23897 |
| API and Mobile Abuse | Flask IDOR, JSON API SQLi, Android APK reverse engineering with jadx |

### Privilege Escalation

| Skill Area | Tools and Techniques |
|------------|---------------------|
| Linux Privilege Escalation | SUID/SGID binary abuse, sudo misconfiguration, GTFOBins shell escapes |
| Sudo Abuse | sudo vim, sudo env, sudo grep, sudo dd, sudo cat, sudo perl, custom sudo binaries |
| Cron and File Abuse | Writable scheduled scripts, world writable binaries, file ownership hijacking |
| Credential Reuse | Database config credentials, base64 and base85 encoded secrets, /etc/passwd takeover |
| Lateral Movement | Multi hop sudo chains, su brute forcing, SSH key disclosure via LFI |

### Post-Exploitation and Documentation

| Skill Area | Tools and Techniques |
|------------|---------------------|
| Evidence Collection | Screenshot capture, terminal logging, artifact preservation |
| Attack Path Mapping | Attack chain documentation, vulnerability correlation |
| Technical Writing | Executive summary generation, methodology documentation, reproducible reporting |
| Creative Techniques | Port knocking, riddle solving, OSINT based password generation, themed challenges |

---

## Machines Inventory

### Completed Virtual Machines (74)

<details>
<summary><b>View Complete Machine List</b></summary>

| Machine | Author | Difficulty | Primary Attack Vectors |
|---------|--------|------------|------------------------|
| aguademayo | The Hackers Labs | Easy | Brainfuck encoded credential, SSH access, privilege escalation chain |
| aidor | El Pingüino de Mario | Easy | Flask IDOR on /dashboard, ffuf id parameter enumeration |
| allien | Luisillo_o | Easy | SMB anonymous share, JWT with RSA key, lateral movement |
| amor | Romabri | Easy | Web reconnaissance, weak security practices, credential recovery |
| anonymouspingu | El Pingüino de Mario | Easy | Anonymous FTP upload, PHP webshell, sudo chain via man nmap chown |
| apibase | El Pingüino de Mario | Easy | Flask Werkzeug REST API, /add SQL injection |
| apkadmin | El Pingüino de Mario | Easy | Android APK reverse engineering, jadx, AdminActivity intent extra |
| autoescuela | mikisbd | Easy | SQL injection in login form, credential recovery |
| backend | 4bytes | Easy | SQL injection, password dumping |
| balufood | El Pingüino de Mario | Easy | Flask app, /login and /admin panels, privilege escalation |
| balulero | El Pingüino de Mario | Easy | Exposed .env file via JS source, privilege escalation |
| bicho | Trr0r | Easy | WordPress via vhost bicho.dl, WPScan, privilege escalation |
| borazuwarahctf | borazuwarah | Very Easy | Web reconnaissance, steganography, brute force credentials |
| breakmyssh | El Pingüino de Mario | Very Easy | OpenSSH 7.7 CVE-2018-15473 user enum, Hydra, root SSH |
| bypassme | RSA | Easy | PHP login SQLi bypass, privilege escalation |
| candy | Luisillo_o | Easy | Joomla robots.txt creds, template RCE, sudo dd privilege escalation |
| chocolatelovers | El Pingüino de Mario | Easy | Nibbleblog CMS exploitation, privilege escalation |
| consolelog | El Pingüino de Mario | Easy | Express and SSH, debug token leak, proxy abuse |
| dockerlabs | El Pingüino de Mario | Easy | Upload workflow, weak upload controls |
| duque | jes507 | Easy | SQLi in /bills/index.php, credential dump |
| ejotapete | El Pingüino de Mario | Easy | Drupal 7 or 8 installation, Drupalgeddon exploitation |
| elevator | beafn28 | Easy | File upload mechanism, Scooby Doo themed, privilege escalation |
| escolares | Luisillo_o | Easy | Source review, WordPress, user enumeration |
| extraviado | Hack_Viper | Easy | Base64 credential chain, hidden files, riddle for root password |
| file | Scuffito y Jul3n-dot | Easy | Anonymous FTP, upload filter bypass, su bruteforce, stego, four hop privesc |
| findyourstyle | El Pingüino de Mario | Easy | Drupal 8 Drupalgeddon 2 RCE, db config creds, sudo grep abuse |
| firsthacking | El Pingüino de Mario | Very Easy | vsftpd 2.3.4 backdoor CVE-2011-2523 |
| forbiddenhack | d1se0 | Easy | 403 Referer bypass, LFI to PHP filter RCE, custom binary furb |
| galeria | Raul A | Easy | Anonymous FTP exposing web root, .htaccess, Gallery application |
| gotham | TheBat | Easy | JWT auth, default creds guest guest, secret batman crack |
| grooti | Grooti16 | Easy | Web info leak, exposed MySQL, privilege escalation |
| hedgehog | AnkbNikas | Very Easy | Web root word tails as username hint, SSH access |
| hiddencat | El Pingüino de Mario | Easy | Tomcat AJP 8009, Ghostcat LFI, privilege escalation |
| influencerhate | El Pingüino de Mario | Easy | HTTP Basic Auth, Hydra default creds, privilege escalation |
| injection | El Pingüino de Mario | Easy | SQLi in PHP login name parameter, authentication bypass |
| internal | buda-sys | Easy | SysVault Backup Manager, vhost enum, Directory Inspector LFI |
| internship | s1egfr1ed | Easy | SQLi, password cracking, vhost, stego, cron lateral, sudo vim |
| jenkhack | d1se0 | Easy | Jenkins Groovy console RCE, base85 creds, custom sudo binary |
| kmspwned | b4d1t | Easy | ServiCloud, SQLi in JSON API search, privilege escalation |
| library | El Pingüino de Mario | Easy | index.php token, privilege escalation |
| los40ladrones | firstatack | Easy | Port knocking, three port knock, Hydra, SUID bash |
| mirame | maciiii___ | Easy | SQLi login, sqlmap extraction, privilege escalation |
| move | El Pingüino de Mario | Easy | Grafana 8.3.0 CVE-2021-43798 dir traversal, misconfigured web server |
| nodeclimb | El Pingüino de Mario | Easy | Anonymous FTP, zip2john and john cracking, privilege escalation |
| obsession | Juan | Very Easy | Anonymous FTP, plaintext files, privilege escalation |
| paradise | kaikoperez | Easy | Image directory, edited and original photo, media artifact abuse |
| patriaquerida | JuanR | Easy | LFI, credential note, lateral movement, SUID Python privesc |
| picadilly | kaikoperez | Easy | Exposed backup.txt encoded credential, hint, privilege escalation |
| pingctf | borazuwarah | Easy | Apache ping utility, OS command injection, Python reverse shell |
| pinguinazo | El Pingüino de Mario | Easy | Flask, PinguNombre SSTI, privilege escalation |
| pkgpoison | RSA | Easy | Public note.txt dev credentials, privilege escalation |
| pn | El Pingüino de Mario | Easy | Tomcat 9.0.88, Manager and Host Manager default credentials |
| pntopntobarra | maciiii___ | Easy | LFI to SSH private key disclosure, sudo env root escalation |
| pressenter | d1se0 | Easy | WordPress, WPScan, XML-RPC bruteforce, functions.php webshell, sudo cat shadow |
| psycho | Luisillo_o | Easy | LFI parameter fuzz, SSH key, sudo perl, Python script hijack |
| redirection | El Pingüino de Mario | Easy | Three open redirect labs, real privilege escalation chain |
| reflection | El Pingüino de Mario | Easy | Four XSS labs, privilege escalation chain |
| secretjenkins | El Pingüino de Mario | Easy | Jenkins CVE-2024-23897 file disclosure, privilege escalation |
| showtime | maciiii___ | Easy | SQLi, three hop lateral movement and privilege escalation |
| stellarjwt | Alv-fh | Easy | OSINT password generation, NASA trivia, three hop sudo, /etc/passwd takeover |
| tproot | d1se0 | Very Easy | vsftpd 2.3.4 backdoor trigger, privilege escalation |
| trust | El Pingüino de Mario | Very Easy | Apache and SSH, feroxbuster discovery, privilege escalation |
| upload | El Pingüino de Mario | Easy | upload.php handler, sudo misconfiguration |
| vacaciones | Romabri | Very Easy | curl reveals HTTP body, privilege escalation |
| verdejo | The Hackers Labs | Easy | Jinja2 SSTI on Flask, GTFOBins sudo, encrypted SSH key recovery |
| vulnvault | d1se0 | Easy | Report generator, upload interface, legacy path exploitation |
| walking dead | JuanR | Easy | Walking Dead themed CTF, hidden directory, concealed PHP webshell |
| walkingcms | El Pingüino de Mario | Easy | WordPress /wordpress/, WPScan author mario, privilege escalation |
| wargames | kaikoperez | Easy | W.O.P.R. themed, FTP SSH HTTP SMB, privilege escalation |
| whereismywebshell | El Pingüino de Mario | Easy | shell.php parameter discovery, privilege escalation |
| whoiam | 0xR0_ | Easy | WordPress, leaked db backup, plugin file upload, three nested sudo |
| winfake | d1se0 | Easy | Fake Windows news portal, password hint, social engineering |
| winterfell | Zunderrub | Easy | Game of Thrones themed SMB, enum4linux users, shared share, sudo privesc |
| pequeñas-mentirosas | beafn28 | Easy | SSH and Apache, credential clue for user a, privilege escalation |

</details>

### Machine Authors

The DockerLabs community contributes machines from a wide range of authors. The most prolific contributors are:

| Author | Machines Authored | Count |
|--------|-------------------|-------|
| El Pingüino de Mario | aidor, anonymouspingu, apibase, apkadmin, balufood, balulero, breakmyssh, chocolatelovers, consolelog, dockerlabs, ejotapete, findyourstyle, hiddencat, influencerhate, injection, library, move, nodeclimb, pn, pinguinazo, redirection, reflection, secretjenkins, trust, upload, walkingcms, whereismywebshell, and more | 28 |
| d1se0 | forbiddenhack, jenkhack, pressenter, tproot, vulnvault, winfake | 6 |
| Luisillo_o | allien, candy, escolares, psycho | 4 |
| maciiii___ | mirame, pntopntobarra, showtime | 3 |
| kaikoperez | paradise, picadilly, wargames | 3 |
| The Hackers Labs | aguademayo, verdejo | 2 |
| RSA | bypassme, pkgpoison | 2 |
| Romabri | amor, vacaciones | 2 |
| JuanR | patriaquerida, walking dead | 2 |
| buda-sys | internal | 2 |
| borazuwarah | borazuwarahctf, pingctf | 2 |
| beafn28 | elevator, pequeñas-mentirosas | 2 |
| Others | single machine each (Zunderrub, Trr0r, TheBat, Scuffito y Jul3n-dot, s1egfr1ed, Raul A, mikisbd, Juan, jes507, Hack_Viper, Grooti16, firstatack, b4d1t, AnkbNikas, Alv-fh, 4bytes, 0xR0_) | 17 |

**Notable Machine Families and Themes:**

- **CMS Exploitation Machines**: WordPress (bicho, pressenter, walkingcms, whoiam), Joomla (candy), Drupal (ejotapete, findyourstyle), Nibbleblog (chocolatelovers)
- **Service Exploitation Machines**: Jenkins (jenkhack, secretjenkins), Tomcat (hiddencat, pn), Grafana (move), vsftpd backdoor (firsthacking, tproot)
- **Themed Challenges**: winterfell (Game of Thrones), wargames (W.O.P.R.), walking dead, elevator (Scooby Doo)
- **Educational Lab Machines**: redirection (open redirect labs), reflection (XSS labs)

---

## Learning Outcomes and Demonstrated Competencies

This portfolio demonstrates systematic competency development across core penetration testing domains.

### Strategic Capabilities

- **Attack Surface Analysis**: Comprehensive enumeration of network services, web applications, and system configurations
- **Vulnerability Assessment**: Identification and prioritization of exploitable weaknesses
- **Exploit Chain Development**: Construction of multi-stage attack paths from initial access to privilege escalation
- **Post-Exploitation Operations**: Evidence collection, credential harvesting, and attack path documentation

### Technical Proficiencies

- **Web Application Security**: SQL injection, local and remote file inclusion, file upload bypass, command injection, server side template injection, and authentication bypass
- **Linux Security**: Permission models, SUID abuse, sudo misconfiguration, GTFOBins shell escapes, and multi hop lateral movement
- **Service Security**: Jenkins, Tomcat, Grafana, WordPress, Drupal, Joomla, and legacy daemons like vsftpd and OpenSSH
- **Authentication and Crypto**: JWT cracking, IDOR abuse, base64 and base85 encoded secrets, and password store recovery

### Professional Standards

- **Documentation Excellence**: Clear, reproducible, and professionally formatted technical reports
- **Methodological Rigor**: Systematic approach to vulnerability discovery and exploitation
- **Attack Chain Analysis**: Complete traceability from reconnaissance through privilege escalation
- **Evidence Preservation**: Synchronized terminal logs and visual documentation

---

## Navigation Guide

### For Security Professionals

To evaluate technical proficiency:

1. Review the Technical Skills Matrix for capability overview
2. Examine writeups from the machine directories for methodology assessment
3. Analyze the CMS exploitation machines for content management system expertise
4. Review the Attack Chain Summaries to understand exploitation sequences

### For Technical Recruiters

To assess candidate suitability:

1. Examine the Portfolio Philosophy section for professional approach
2. Review 2 to 3 sample machine writeups for documentation quality
3. Evaluate the Technical Skills Matrix against position requirements
4. Assess the scale and consistency of the portfolio (74 machines documented)

### For Students and CTF Players

To learn penetration testing techniques:

1. Start with Very Easy machines such as breakmyssh, firsthacking, or trust
2. Progress to Easy CMS machines for structured web exploitation learning
3. Study the Attack Chain Summaries to understand end to end exploitation sequences
4. Compare the deployment and scanning steps to learn Docker based lab methodology

---

## Documentation Standards

All writeups in this repository adhere to the following standards:

- **Structured Format**: Consistent use of Executive Summary, Reconnaissance, Initial Access, optional Lateral Movement, Privilege Escalation, and Attack Chain Summary sections
- **Command Reproducibility**: Complete command syntax with full output preservation, never truncated
- **Visual Evidence**: Screenshots embedded inline at relevant technical steps using original filenames
- **Technical Accuracy**: Validated exploitation paths with reproducible results
- **Professional Presentation**: Clean Markdown formatting suitable for technical documentation

---

## Legal and Ethical Disclaimer

**All activities documented in this repository were conducted in authorized laboratory environments.**

This repository is intended exclusively for:

- Educational purposes and professional skill development
- Portfolio demonstration for employment opportunities
- Security research within controlled environments (DockerLabs platform, locally deployed containers)

**Legal Notice:**

Unauthorized access to computer systems is illegal under applicable laws including but not limited to:

- Computer Fraud and Abuse Act (CFAA), United States
- Computer Misuse Act, United Kingdom
- Ley Orgánica de Protección de Datos y Garantía de los Derechos Digitales, Spain
- Cybercrime laws in various jurisdictions

The techniques documented in this repository must only be applied:

- On systems you own
- On systems where you possess explicit written authorization
- In legitimate Capture the Flag or laboratory environments

**Liability Disclaimer:**

The author assumes no liability for misuse of the information contained in this repository. Users are solely responsible for ensuring compliance with applicable laws and regulations. Any application of these techniques without proper authorization may result in criminal prosecution.

---

## Contributing and Collaboration

While this repository serves primarily as a personal portfolio, technical feedback is welcomed:

- **Technical Corrections**: Report errors or inaccuracies through issue tracking
- **Methodology Discussions**: Share alternative approaches or optimization techniques
- **Documentation Improvements**: Suggest enhancements to reporting clarity or structure

For professional inquiries or collaboration proposals, contact information can be provided upon request.

---

## Repository Roadmap

### Current Status (August 2026)

- 74 machine writeups completed with full documentation
- Standardized executive summary format implemented across all reports
- Multimodal documentation workflow operational
- Coverage spanning web, service, CMS, and privilege escalation technique categories

### Planned Enhancements

- Extension of the machine inventory as new DockerLabs machines are released
- Cross-machine vulnerability trend analysis and pattern documentation
- Refinement of reporting consistency across older entries
- Addition of higher difficulty machines as they are published on the platform

---

## Acknowledgments

- **DockerLabs Platform**: For providing an accessible, container based learning environment
- **Machine Authors**: El Pingüino de Mario, d1se0, Luisillo_o, maciiii___, kaikoperez, The Hackers Labs, and the many community contributors for designing realistic and educational security scenarios
- **Security Community**: For continuous knowledge sharing and collaborative research

---

<div align="center">

![Maintained](https://img.shields.io/badge/Maintained-Yes-brightgreen?style=flat-square)
![Last Updated](https://img.shields.io/badge/Last_Updated-August_2026-blue?style=flat-square)

**Professional Penetration Testing Documentation**

</div>
