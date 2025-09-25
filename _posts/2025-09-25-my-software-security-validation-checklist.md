---
layout: post
title: "My Security Software Validation Checklist"
date: 2025-09-25 16:00:00 +0000
categories: [security, osint]
tags: [sca, osint, sbom]
author: alphajerboa
excerpt: "How to evaluate software trustworthiness and security"
---

My go-to checklist for security evaluations when a colleague requests installing some new software in the enterprise

## Product Quality Evaluation

### Company, Community, and Committer Reputation Assessment

**Open Source Intelligence (OSINT) Research**
- Research the development team's background and security track record
- Analyze community engagement and responsiveness to security issues
- Evaluate maintainer reputation and contribution history (books, events, ....)
- Check for any past security incidents or controversies
- Review funding sources and potential conflicts of interest

### Code Quality Evaluation

#### Static Code Analysis

*For Open Source Software:*
- **Manual Code Review**: Conduct targeted manual review of critical components
- **Automated SAST Tools**: Use Static Application Security Testing tools
- **CI/CD Integration Analysis**: 
  - Examine GitHub workflows (`.github/workflows/*.yml`) for security configurations
  - Verify automated security checks are properly implemented
- **AI-Assisted Security Audit**: Leverage Large Language Models to identify:
  - Telemetry and cloud-export hooks
  - Suspicious data collection mechanisms  
  - Obfuscated code patterns
  - High entropy strings that may indicate hidden functionality
  - Hardcoded credentials or configuration issues

*For Binary/Proprietary Software:*
- **Reverse Engineering Analysis**: Use debuggers such as IDA Pro or Ghidra
- **Binary Analysis**: Examine executable structure and embedded resources
- **Runtime Behavior Analysis**: Monitor system calls and resource access

**Software Composition Analysis (SCA)**
- **Dependencies Mapping**: Generate comprehensive dependency graphs
- **Software Bill of Materials (SBOM)**:
  - For GitHub projects: Check submodules and utilize Insights tab
  - Retrive SBOM and upload SBOM to SCA platforms for vulnerability scanning
- **Third-party Component Assessment**: Evaluate security posture of all dependencies
- **License Compliance Review**: Ensure dependency licenses align with organizational policies


#### Dynamic Code Analysis

**Online Malware Analysis and Sandboxing**
- **VirusTotal** (https://www.virustotal.com/gui/): Submit binaries for multi-engine analysis
- **AlienVault OTX** (https://otx.alienvault.com/submissions/list): Analyze files under 100MB
- **Cuckoo Sandbox** (https://cuckoo.cert.ee/): Automated malware analysis
- **Triage** (https://tria.ge/): Advanced malware analysis platform
- **Hybrid Analysis** (https://www.hybrid-analysis.com/): Comprehensive behavioral analysis

For thorough analysis, install software in vanilla operating systems with OTX endpoint agents to monitor behavioral patterns and network communications.

**On-Premises Custom Sandboxing**
- **System Call Tracing**: Use `strace` to monitor system interactions
- **Network Analysis**: Deploy `tcpdump` and `iptables` for traffic inspection
- **Proxy Analysis**: Implement `mitmproxy` for HTTPS traffic examination
- **Behavioral Monitoring**: Track file system, registry, and network changes
- **Resource Usage Analysis**: Monitor CPU, memory, and disk usage patterns

**Black-box Penetration Testing**
- Conduct external security assessments without source code access
- Test authentication mechanisms and session management
- Evaluate input validation and injection vulnerabilities
- Assess authorization controls and privilege escalation risks

## Secure Development Guidelines Compliance

**GitHub Security Features Review**
- Examine the Security tab for vulnerability alerts and advisories
- Check for security policy documentation
- Verify dependency vulnerability scanning is enabled
- Review code scanning alerts and remediation status
- Assess secret scanning configurations and findings

## External Security Intelligence

### Standards, Regulations, and Certification Compliance

- **Industry Certifications**: Verify SOC 2, ISO 27001, or relevant compliance certifications
- **Regulatory Alignment**: Ensure compliance with GDPR, CCPA, HIPAA, or applicable regulations
- **Security Standards**: Check adherence to OWASP, NIST, or industry-specific security frameworks
- **Audit Reports**: Review available third-party security assessments

### Bug Bounty Programs

- **Active Programs**: Check Bugcrowd (https://www.bugcrowd.com/bug-bounty-list/) for active bounty programs
- **Historical Findings**: Review past vulnerability disclosures and remediation timelines
- **Researcher Engagement**: Evaluate vendor responsiveness to security researcher reports
- **Scope and Rewards**: Assess the comprehensiveness of bug bounty program scope

### External Security Assessments

- **Penetration Testing Reports**: Request and review recent third-party penetration test results
- **Security Audit Documentation**: Examine formal security audits by reputable firms
- **Contractor Assessments**: Evaluate security reviews conducted by external specialists
- **Remediation Evidence**: Verify that identified issues have been properly addressed

### Security Advisory and Vulnerability Research

- **Known Vulnerabilities**: Search CVE databases for reported security issues
- **Exploit Availability**: Check for publicly available exploits or proof-of-concept code
- **Security Advisories**: Review vendor security advisories and patch release notes
- **Threat Intelligence**: Consult threat intelligence feeds for indicators of compromise
- **Zero-day Research**: Monitor security research publications for emerging threats

### Data Privacy Policy Evaluation

- **Privacy Documentation**: Review comprehensive privacy policies and data handling practices
- **Data Collection Scope**: Assess what data is collected, processed, and stored
- **Third-party Sharing**: Evaluate data sharing agreements with external parties
- **User Rights**: Verify support for data subject rights (access, deletion, portability)
- **Retention Policies**: Examine data retention and deletion procedures
- **Geographic Considerations**: Assess data residency and cross-border transfer controls
