# ANIICS Skill

Agent skill for integrating, operating, and extending the ANIICS Cloudflare Workers analytics service.

## What is ANIICS?

ANIICS is a Cloudflare Workers-based analytics service that provides event tracking and dashboard capabilities using Analytics Engine.

## What This Skill Does

This skill provides AI agents with guidance for:

- Integrating ANIICS tracking into applications
- Working with the event ingestion API (`/api/v1/p`)
- Securing and querying the `/mothersh1p` dashboard
- Extending the Analytics Engine schema
- Modifying the ANIICS service implementation

## Install

```bash
npx skills add https://github.com/aniics/aniics-skill --skill aniics -g -y
```

## Usage

Once installed, agents will automatically use this skill when working on ANIICS-related tasks, including app instrumentation, schema changes, dashboard modifications, and service debugging.

## License

MIT License - see LICENSE file for details.

## Maintenance

This repository is generated from the private ANIICS application repository. Changes should be made in the private repo and synced here.
