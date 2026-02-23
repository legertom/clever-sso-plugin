# Clever SSO Plugin

This repository contains a Claude Code skill for integrating web applications with Clever SSO (Single Sign-On) for K-12 education.

## What This Is

A Claude Code skill (`/clever-sso`) that guides developers through the complete process of adding Clever SSO to their web application. It handles:

- Detecting the user's tech stack automatically
- Generating complete, working OAuth 2.0 integration code
- Database schema changes for storing Clever user data
- Login button implementation with official Clever assets
- Error handling for all Clever-specific edge cases
- Testing guidance with Clever's sandbox environment
- Certification preparation checklist

## Project Structure

```
.claude/skills/clever-sso/
  SKILL.md       - Main skill instructions (Claude reads this)
  reference.md   - Detailed technical reference (endpoints, responses, tokens)
  examples.md    - Complete framework-specific implementations
```

## Key Clever SSO Concepts

- Clever uses OAuth 2.0 authorization code grant flow
- Tokens expire after 24 hours with no refresh mechanism
- Clever ID is the ONLY reliable user identifier (emails are unverified)
- Production redirect URIs must be HTTPS
- District SSO provides basic user data; Library SSO adds classroom data
- Certification is required before going live with districts

## Development

To test this skill, run `/clever-sso` in Claude Code from any web project directory.
