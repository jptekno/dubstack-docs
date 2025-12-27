---
title: Major Homelab Infrastructure Updates - September 2025
date: 2025-09-15 10:00 +1000
categories: [homelab, infrastructure]
tags: [truenas, storage, kubernetes, automation, documentation]
---

# Major Homelab Infrastructure Updates - September 2025

## 🎯 Overview

This post covers the significant infrastructure improvements made to the OztekLab homelab during September 2025. These updates focus on storage optimization, documentation organization, and workflow automation to create a more robust and maintainable infrastructure.

---

## 🔧 TrueNAS Storage Optimization

### Problem Solved: HGST Drive DIF Compatibility Issues

**Background:** Our HGST HUH721212AL4200 12TB enterprise drives were causing TrueNAS pool creation failures due to DIF (Data Integrity Field) protection being enabled by default.

**Solution Implemented:**
- Created automated monitoring scripts for `sg_format` operations
- Developed DIF-disabled 4K sector formatting: `--size=4096 --fmtpinfo=0`
- Implemented comprehensive drive status monitoring with automatic recovery

### Key Achievements:
✅ **14 enterprise drives** being formatted to 4K sectors with DIF disabled  
✅ **Automated monitoring scripts** with real-time progress tracking  
✅ **Comprehensive documentation** created in Outline knowledge base  
✅ **Version-controlled scripts** in `ozteklab/infra` repository  


### Technical Details:
```bash
# Critical formatting command with DIF disabled
sg_format --format --size=4096 --fmtpinfo=0 --count=1 /dev/sdX

# Drive mapping (NAS-02)
- sda: System drive (TrueNAS OS)
- sdb: Special case (excluded)  
- sdc-sdp: 14 drives targeted for 4K DIF-disabled formatting
```

---

## 📚 Infrastructure Documentation Overhaul

### Unified Documentation Repository

**Repository:** `https://git.ozteklab.com/ozteklab/infra`

**Accomplished:**
✅ **Removed 2 unnecessary collections** (Test, tetst)  
✅ **Deleted 8+ duplicate/empty documents**  
✅ **Reorganized 20+ documents** into proper collections  
✅ **Consolidated academic research** in dedicated collection  

**Key Collections Organized:**
- **TrueNAS:** Storage configuration and SG_FORMAT scripts
- **Kubernetes:** K3s, Traefik, Rancher comprehensive guides  
- **N8N:** Workflow automation documentation


---

## 🤖 Automation & Workflow Enhancements

### N8N MCP Wrapper Integration

**Achievement:** Successfully integrated N8N with MCPO wrapper providing 48+ functions for:
- **Outline API:** Complete document management
- **Workflow Automation:** Advanced N8N operations
- **Infrastructure Management:** Cross-platform orchestration

### Monitoring Scripts Development

**SG_FORMAT Monitoring System:**
- **Real-time Progress Tracking:** Percentage-based monitoring
- **Automatic Drive Recovery:** Reset stuck drives automatically  
- **Concurrent Operation Management:** Limit to 6 drives to prevent overload
- **Smart Status Indicators:** Visual feedback with colored terminal output

---

## 📊 Current Infrastructure Status

**TrueNAS (nas-02.internal.ozteklab.com):**
- **Status:** Active formatting of 14x 12TB HGST drives
- **Progress:** 1-2 drives completing 4K DIF-disabled formatting
- **Expected Completion:** 3-7 days for all drives

**Edge Docker Host (192.168.70.10):**
- **Traefik:** Load balancing and SSL termination
- **Jekyll:** Homelab documentation website (ozteklab.com)
- **Services:** Running under `/srv` directory structure


**K3s Cluster:**
- **Ingress:** Dual Traefik setup (internal/external)
- **Storage:** Longhorn distributed storage
- **Management:** Rancher UI for cluster administration

---

## 🔗 Resources & References

### Key Commands Reference:
```bash
# Check formatting progress
./quick_status.sh

# Start continuous monitoring  
./sg_format_monitor.sh

# Validate drive formatting
sg_format --count=1 /dev/sdX | grep "Block size"
```

---

## 📝 Conclusion

The September 2025 infrastructure updates represent a significant leap forward in homelab maturity and reliability. The resolution of the HGST drive DIF compatibility issues, combined with comprehensive documentation organization and automation enhancements, creates a solid foundation for future growth.

**Key Success Metrics:**
- 🎯 **Problem Resolution:** Critical storage issue identified and solved
- 📚 **Documentation Quality:** Knowledge base cleaned and organized  
- 🤖 **Automation Level:** Monitoring and recovery scripts implemented
- 🔄 **Version Control:** Complete infrastructure tracking in git

These improvements ensure the OztekLab infrastructure is well-documented, properly monitored, and ready for production workloads.

---

*Last Updated: September 15, 2025*
*Infrastructure Repository: [ozteklab/infra](https://git.ozteklab.com/ozteklab/infra)*

---

## 🧪 API Playground Test

Test the Ragtronic Chat API directly from this post:

<Snippet file="api/ragtronic-api-chat-chat-post.mdx" />
