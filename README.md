# wait-i-have-to-secure-WHAT

A practical security guide for professionals transitioning from cloud computing to high-performance computing (HPC) environments.

## What is this?

If you've built your career securing cloud infrastructure and suddenly find yourself responsible for a supercomputer, this guide is for you. HPC security isn't fundamentally different from cloud security—but it *is* different enough to be confusing.

This repository contains:
- **HPC to Cloud Translation Guide**: Maps every major HPC concept to cloud computing equivalents
- **6-Month Study Plan**: Structured learning path for ramping up on HPC security
- Practical security considerations for HPC-specific technologies

## Who is this for?

- Cloud security professionals moving into HPC environments
- ISSOs/security engineers taking on supercomputing responsibilities
- Anyone who knows AWS/Azure but not Slurm/GPFS
- People who thought "how different can it be?" and then found out

## The Philosophy

You don't need to become an HPC expert to secure HPC systems effectively. You need to:
1. Understand how HPC differs from cloud computing
2. Apply your existing security expertise appropriately
3. Ask the right questions (even if they seem "basic")

Your "outsider perspective" is actually valuable—you won't accept "we've always done it this way" as a security justification.

## What's Inside

### [HPC to Cloud Translation Guide](hpc_to_cloud_translation_guide.md)
Maps HPC concepts (compute nodes, job schedulers, parallel filesystems) to their cloud equivalents, then explains what's different and why it matters for security.

**Topics covered:**
- Compute models: Batch jobs vs. persistent services
- Storage: Parallel filesystems vs. S3/EFS
- Networking: InfiniBand vs. VPCs
- Security primitives: Unix permissions vs. IAM
- Threat models: Insider threats vs. external attacks

### [HPC Security Study Plan](hpc_security_study_plan.md)
A 6-month learning roadmap covering:
- HPC fundamentals
- Job schedulers and resource management
- Storage and data security
- Network architecture
- Monitoring and incident response
- Compliance and vulnerability management

Designed for ADHD-friendly learning: hands-on first, practical application, and connecting new concepts to existing knowledge.

## Why This Exists

Because when you search for "HPC security for cloud professionals," you get:
- Academic papers written for people who already understand HPC
- Vendor documentation that assumes you know the jargon
- Conference presentations for HPC experts

There wasn't a practical "I know cloud, teach me HPC security" guide. So I made one.

## Contributing

Found something confusing? Have a better cloud analogy? Spotted an error? PRs welcome.

## Disclaimer

This guide reflects real-world experience securing multi-petabyte scientific computing environments. Your mileage may vary. Not all HPC centers operate the same way, but the core concepts translate.

## License

MIT License - use this however it helps you.

## Acknowledgments

Thanks to every HPC professional who patiently explained "yes, it's like that cloud thing... but different."

---

*"The best time to learn HPC security was before you were responsible for it. The second best time is now."*
