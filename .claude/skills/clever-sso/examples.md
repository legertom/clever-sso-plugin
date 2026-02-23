# Clever SSO Integration Examples

Framework-specific implementation examples for common stacks. These are complete, working implementations — not snippets.

## Next.js (App Router) with NextAuth.js

### 1. Environment Variables (`.env.local`)
```
CLEVER_CLIENT_ID=your_client_id_here
CLEVER_CLIENT_SECRET=your_client_secret_here
NEXTAUTH_URL=http://localhost:3000
NEXTAUTH_SECRET=generate-a-random-secret-here
```

### 2. Clever Provider (`lib/clever-provider.ts`)
```typescript
import type { OAuthConfig, OAuthUserConfig } from "next-auth/providers/oauth";

interface CleverProfile {
  id: string;
  type: string;
  district: string;
  email?: string;
  name: {
    first: string;
    last: string;
  };
  roles: Record<string, unknown>;
}

export default function CleverProvider(
  config: OAuthUserConfig<CleverProfile>
): OAuthConfig<CleverProfile> {
  return {
    id: "clever",
    name: "Clever",
    type: "oauth",
    authorization: {
      url: "https://clever.com/oauth/authorize",
      params: { response_type: "code" },
    },
    token: "https://clever.com/oauth/tokens",
    userinfo: {
      url: "https://api.clever.com/v3.0/me",
      async request({ tokens }) {
        // First get the /me endpoint to find user type and ID
        const meRes = await fetch("https://api.clever.com/v3.0/me", {
          headers: {
            Authorization: `Bearer ${tokens.access_token}`,
            Accept: "application/json",
          },
        });
        const me = await meRes.json();

        // Then get full user details
        const userRes = await fetch(
          `https://api.clever.com/v3.0/users/${me.data.id}`,
          {
            headers: {
              Authorization: `Bearer ${tokens.access_token}`,
              Accept: "application/json",
            },
          }
        );
        const user = await userRes.json();

        return {
          id: me.data.id,
          type: me.type,
          district: me.data.district,
          email: user.data.email,
          name: user.data.name,
          roles: user.data.roles,
        };
      },
    },
    profile(profile) {
      return {
        id: profile.id,
        name: `${profile.name.first} ${profile.name.last}`,
        email: profile.email ?? null,
      };
    },
    style: {
      brandColor: "#4274f6",
      logo: "https://files.readme.io/d1a4142-blue-liwc.svg",
    },
    ...config,
  };
}
```

### 3. Auth Config (`lib/auth.ts`)
```typescript
import NextAuth from "next-auth";
import CleverProvider from "./clever-provider";

export const { handlers, signIn, signOut, auth } = NextAuth({
  providers: [
    CleverProvider({
      clientId: process.env.CLEVER_CLIENT_ID!,
      clientSecret: process.env.CLEVER_CLIENT_SECRET!,
    }),
  ],
  callbacks: {
    async signIn({ user, account, profile }) {
      // profile contains the CleverProfile data
      // Here you would match/create the user in your database
      // using the Clever ID as primary identifier
      return true;
    },
    async jwt({ token, account, profile }) {
      if (account && profile) {
        token.cleverId = (profile as any).id;
        token.cleverType = (profile as any).type;
        token.cleverDistrict = (profile as any).district;
      }
      return token;
    },
    async session({ session, token }) {
      session.user.cleverId = token.cleverId as string;
      session.user.cleverType = token.cleverType as string;
      session.user.cleverDistrict = token.cleverDistrict as string;
      return session;
    },
  },
});
```

### 4. Login Button Component (`components/clever-login-button.tsx`)
```tsx
"use client";

import { signIn } from "next-auth/react";

export function CleverLoginButton() {
  return (
    <button
      onClick={() => signIn("clever")}
      style={{
        background: "none",
        border: "none",
        cursor: "pointer",
        padding: 0,
      }}
    >
      <img
        src="https://files.readme.io/d1a4142-blue-liwc.svg"
        alt="Log in with Clever"
        height={40}
      />
    </button>
  );
}
```

---

## Express.js with Passport

### 1. Environment Variables (`.env`)
```
CLEVER_CLIENT_ID=your_client_id_here
CLEVER_CLIENT_SECRET=your_client_secret_here
CLEVER_REDIRECT_URI=http://localhost:3000/auth/clever/callback
SESSION_SECRET=generate-a-random-secret-here
```

### 2. Clever Passport Strategy (`auth/clever-strategy.js`)
```javascript
const OAuth2Strategy = require("passport-oauth2");

class CleverStrategy extends OAuth2Strategy {
  constructor(options, verify) {
    super(
      {
        authorizationURL: "https://clever.com/oauth/authorize",
        tokenURL: "https://clever.com/oauth/tokens",
        clientID: options.clientID,
        clientSecret: options.clientSecret,
        callbackURL: options.callbackURL,
      },
      verify
    );
    this.name = "clever";
  }

  async userProfile(accessToken, done) {
    try {
      // Get /me first
      const meRes = await fetch("https://api.clever.com/v3.0/me", {
        headers: {
          Authorization: `Bearer ${accessToken}`,
          Accept: "application/json",
        },
      });
      const me = await meRes.json();

      // Get full user details
      const userRes = await fetch(
        `https://api.clever.com/v3.0/users/${me.data.id}`,
        {
          headers: {
            Authorization: `Bearer ${accessToken}`,
            Accept: "application/json",
          },
        }
      );
      const user = await userRes.json();

      const profile = {
        id: me.data.id,
        type: me.type,
        district: me.data.district,
        displayName: `${user.data.name.first} ${user.data.name.last}`,
        name: user.data.name,
        email: user.data.email || null,
        roles: user.data.roles,
      };

      done(null, profile);
    } catch (err) {
      done(err);
    }
  }
}

module.exports = CleverStrategy;
```

### 3. Auth Routes (`routes/auth.js`)
```javascript
const express = require("express");
const passport = require("passport");
const router = express.Router();

// Initiate Clever login
router.get("/clever", passport.authenticate("clever"));

// Clever callback
router.get(
  "/clever/callback",
  passport.authenticate("clever", { failureRedirect: "/login?error=clever" }),
  (req, res) => {
    res.redirect("/dashboard");
  }
);

// Handle Clever errors (unauthorized district, etc.)
router.get("/clever/callback", (req, res) => {
  if (req.query.error === "unauthorized-user") {
    return res.redirect(
      "/login?error=Your school hasn't set up access yet. Please ask your administrator."
    );
  }
  res.redirect("/login?error=Something went wrong. Please try again.");
});

// Logout (only from your app, NOT from Clever)
router.post("/logout", (req, res) => {
  req.logout((err) => {
    if (err) return res.status(500).send("Logout failed");
    res.redirect("/login");
  });
});

module.exports = router;
```

### 4. App Setup (`app.js` — relevant parts)
```javascript
const passport = require("passport");
const CleverStrategy = require("./auth/clever-strategy");

passport.use(
  new CleverStrategy(
    {
      clientID: process.env.CLEVER_CLIENT_ID,
      clientSecret: process.env.CLEVER_CLIENT_SECRET,
      callbackURL: process.env.CLEVER_REDIRECT_URI,
    },
    async (accessToken, refreshToken, profile, done) => {
      try {
        // Look up user by Clever ID
        let user = await db.users.findOne({ clever_id: profile.id });

        if (!user && profile.email) {
          // Try matching by email
          user = await db.users.findOne({ email: profile.email });
          if (user) {
            // Link Clever ID to existing account
            user.clever_id = profile.id;
            user.clever_district_id = profile.district;
            user.clever_user_type = profile.type;
            await user.save();
          }
        }

        if (!user) {
          // Create new user
          user = await db.users.create({
            clever_id: profile.id,
            clever_district_id: profile.district,
            clever_user_type: profile.type,
            name: profile.displayName,
            email: profile.email,
          });
        }

        done(null, user);
      } catch (err) {
        done(err);
      }
    }
  )
);
```

---

## Python / Flask

### 1. Environment Variables (`.env`)
```
CLEVER_CLIENT_ID=your_client_id_here
CLEVER_CLIENT_SECRET=your_client_secret_here
CLEVER_REDIRECT_URI=http://localhost:5000/auth/clever/callback
SECRET_KEY=generate-a-random-secret-here
```

### 2. Clever Auth Blueprint (`auth/clever.py`)
```python
import base64
import requests
from flask import Blueprint, redirect, request, session, url_for, flash

clever_bp = Blueprint("clever", __name__, url_prefix="/auth/clever")

CLEVER_AUTH_URL = "https://clever.com/oauth/authorize"
CLEVER_TOKEN_URL = "https://clever.com/oauth/tokens"
CLEVER_API_BASE = "https://api.clever.com/v3.0"


@clever_bp.route("/login")
def login():
    """Redirect user to Clever for authentication."""
    params = {
        "response_type": "code",
        "client_id": current_app.config["CLEVER_CLIENT_ID"],
        "redirect_uri": current_app.config["CLEVER_REDIRECT_URI"],
    }
    auth_url = f"{CLEVER_AUTH_URL}?{'&'.join(f'{k}={v}' for k, v in params.items())}"
    return redirect(auth_url)


@clever_bp.route("/callback")
def callback():
    """Handle Clever OAuth callback."""
    # Check for errors
    error = request.args.get("error")
    if error == "unauthorized-user":
        flash("Your school hasn't set up access yet. Please ask your administrator.")
        return redirect(url_for("main.login"))
    if error:
        flash("Something went wrong with Clever login. Please try again.")
        return redirect(url_for("main.login"))

    code = request.args.get("code")
    if not code:
        return redirect(url_for("main.login"))

    # Exchange code for token
    client_id = current_app.config["CLEVER_CLIENT_ID"]
    client_secret = current_app.config["CLEVER_CLIENT_SECRET"]
    credentials = base64.b64encode(f"{client_id}:{client_secret}".encode()).decode()

    token_response = requests.post(
        CLEVER_TOKEN_URL,
        headers={
            "Authorization": f"Basic {credentials}",
            "Content-Type": "application/json",
            "Accept": "application/json",
        },
        json={
            "code": code,
            "grant_type": "authorization_code",
            "redirect_uri": current_app.config["CLEVER_REDIRECT_URI"],
        },
    )

    if not token_response.ok:
        flash("Failed to authenticate with Clever. Please try again.")
        return redirect(url_for("main.login"))

    access_token = token_response.json()["access_token"]
    headers = {"Authorization": f"Bearer {access_token}", "Accept": "application/json"}

    # Get user type and ID
    me_response = requests.get(f"{CLEVER_API_BASE}/me", headers=headers)
    me_data = me_response.json()

    clever_id = me_data["data"]["id"]
    user_type = me_data["type"]
    district_id = me_data["data"]["district"]

    # Get full user details
    user_response = requests.get(f"{CLEVER_API_BASE}/users/{clever_id}", headers=headers)
    user_data = user_response.json()["data"]

    first_name = user_data["name"]["first"]
    last_name = user_data["name"]["last"]
    email = user_data.get("email")

    # Match or create user in your database
    user = User.query.filter_by(clever_id=clever_id).first()

    if not user and email:
        user = User.query.filter_by(email=email).first()
        if user:
            user.clever_id = clever_id
            user.clever_district_id = district_id
            user.clever_user_type = user_type
            db.session.commit()

    if not user:
        user = User(
            clever_id=clever_id,
            clever_district_id=district_id,
            clever_user_type=user_type,
            first_name=first_name,
            last_name=last_name,
            email=email,
        )
        db.session.add(user)
        db.session.commit()

    # Create session
    session["user_id"] = user.id
    return redirect(url_for("main.dashboard"))
```

---

## Database Migration Examples

### PostgreSQL (raw SQL)
```sql
ALTER TABLE users
ADD COLUMN clever_id VARCHAR(255) UNIQUE,
ADD COLUMN clever_district_id VARCHAR(255),
ADD COLUMN clever_user_type VARCHAR(50);

CREATE INDEX idx_users_clever_id ON users(clever_id);
```

### Prisma
```prisma
model User {
  id               String  @id @default(cuid())
  email            String? @unique
  name             String?
  cleverId         String? @unique @map("clever_id")
  cleverDistrictId String? @map("clever_district_id")
  cleverUserType   String? @map("clever_user_type")
  createdAt        DateTime @default(now())
  updatedAt        DateTime @updatedAt

  @@map("users")
}
```

### Drizzle
```typescript
import { pgTable, text, timestamp, uniqueIndex } from "drizzle-orm/pg-core";

export const users = pgTable("users", {
  id: text("id").primaryKey(),
  email: text("email").unique(),
  name: text("name"),
  cleverId: text("clever_id").unique(),
  cleverDistrictId: text("clever_district_id"),
  cleverUserType: text("clever_user_type"),
  createdAt: timestamp("created_at").defaultNow(),
  updatedAt: timestamp("updated_at").defaultNow(),
});
```

### Django
```python
# In your User model or a profile model
class UserProfile(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE)
    clever_id = models.CharField(max_length=255, unique=True, null=True, blank=True)
    clever_district_id = models.CharField(max_length=255, null=True, blank=True)
    clever_user_type = models.CharField(max_length=50, null=True, blank=True)
```

### SQLAlchemy
```python
class User(db.Model):
    __tablename__ = "users"

    id = db.Column(db.Integer, primary_key=True)
    email = db.Column(db.String(255), unique=True, nullable=True)
    name = db.Column(db.String(255))
    clever_id = db.Column(db.String(255), unique=True, nullable=True, index=True)
    clever_district_id = db.Column(db.String(255), nullable=True)
    clever_user_type = db.Column(db.String(50), nullable=True)
```
