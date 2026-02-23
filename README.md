# /clever-sso

A Claude Code skill that gets your app into the Clever ecosystem fast.

Run `/clever-sso` in Claude Code from any web project and it will:

1. **Detect your stack** — Next.js, Express, Django, Flask, Rails, Laravel, whatever
2. **Scaffold everything** — env vars, OAuth routes, database schema, login button
3. **Generate complete working code** — not snippets, not pseudocode, full files
4. **Handle the gotchas** — unverified emails, shared Chromebooks, 60-second code expiry, multi-role users
5. **Prep you for certification** — everything it builds meets Clever's requirements out of the box

## What is Clever?

[Clever](https://clever.com) is the SSO platform used by most K-12 schools in the US. If you're building an EdTech app and want schools to use it, you need Clever SSO. This skill makes that integration trivial.

## Install

Copy the skill into your project:

```bash
# From your project root
mkdir -p .claude/skills
cp -r clever-sso-plugin/.claude/skills/clever-sso .claude/skills/
```

Or clone directly:

```bash
git clone https://github.com/legertom/clever-sso-plugin.git
cp -r clever-sso-plugin/.claude/skills/clever-sso your-project/.claude/skills/
```

## Usage

In Claude Code, from your project directory:

```
/clever-sso
```

Or specify your framework:

```
/clever-sso nextjs
/clever-sso express
/clever-sso django
/clever-sso flask
/clever-sso rails
/clever-sso laravel
```

## What Gets Generated

| Component | Description |
|---|---|
| `.env` / `.env.local` | Clever credentials with exact instructions on where to find them |
| OAuth routes | Authorization, callback, token exchange, user info retrieval |
| User matching | Clever ID lookup, email fallback, new account creation |
| DB migration | `clever_id`, `clever_district_id`, `clever_user_type` columns |
| Login button | Official "Log in with Clever" branded SVG button |
| Error handling | Unauthorized districts, expired codes, missing emails, shared devices |

## Clever SSO in 30 Seconds

Clever uses OAuth 2.0 authorization code flow:

```
User clicks "Log in with Clever"
  → Redirects to clever.com/oauth/authorize
  → User authenticates with Clever
  → Clever redirects back with ?code=AUTH_CODE
  → Your server exchanges code for access_token
  → You call /v3.0/me to get the Clever user ID
  → You call /v3.0/users/{id} for name, email, role
  → Match or create user in your database
  → Done
```

Key things the skill knows that you'd otherwise learn the hard way:
- **Emails are unverified** — never use email as primary identifier, always use Clever ID
- **Tokens expire in 24 hours** with no refresh — use your own sessions
- **Auth codes expire in ~60 seconds** — exchange immediately
- **Schools share Chromebooks** — always override existing sessions on new login
- **No single logout** — only log users out of YOUR app, never Clever

## Skill Structure

```
.claude/skills/clever-sso/
  SKILL.md       — Main skill prompt (what Claude follows)
  reference.md   — API endpoints, response formats, token details
  examples.md    — Complete implementations for Next.js, Express, Flask, + DB examples
```

## Supported Stacks

The skill adapts to whatever it finds in your codebase. Tested examples included for:

- **Next.js** + NextAuth
- **Express** + Passport
- **Flask** + SQLAlchemy
- **Database ORMs**: Prisma, Drizzle, Django, SQLAlchemy, raw SQL

It will generate for any stack — these just have pre-built reference implementations.

## Links

- [Clever Developer Docs](https://dev.clever.com)
- [Clever Developer Dashboard](https://apps.clever.com)
- [Claude Code Skills Docs](https://docs.anthropic.com/en/docs/claude-code/skills)
- [Agent Skills Specification](https://agentskills.io/specification)
