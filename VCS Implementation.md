
# Sprint-2 VCS Implementation

> Interview Revision Notes

---

# Ticket 1: Setup VCS

## What is VCS?
Version Control System (VCS) ek tool hai jo source code ko manage karta hai. Isse code ka history maintain hota hai aur multiple developers ek hi project par safely kaam kar sakte hain.

## Why do we need VCS?
- Code History maintain hoti hai.
- Team Collaboration easy hoti hai.
- Backup & Recovery milti hai.
- Code Review possible hota hai.
- CI/CD integration hoti hai.
- Access Control manage hota hai.

## Architecture

```
Developer
    │
Local Git Repository
    │
Remote Repository (GitHub/GitLab/Bitbucket)
    │
Webhook
    │
Jenkins Pipeline
```

## Internal Working

### git push
- Local code remote repository me upload hota hai.
- Remote repository latest commit save karti hai.

### Webhook
- GitHub/GitLab automatically Jenkins ko notification bhejta hai.
- Jenkins pipeline automatically trigger ho jati hai.

### Authentication
- User login verify hota hai.

### Authorization
- Login ke baad permissions check hoti hain.

## SaaS vs On-Prem

### SaaS
- Fast setup
- No maintenance
- Vendor manage karta hai

### On-Prem
- Full control
- Company khud maintain karti hai
- Banking/Government me zyada use hota hai

## Tool Comparison

### GitHub
- Best Community
- Best Integrations

### GitLab
- Built-in CI/CD
- Built-in Security

### Bitbucket
- Best Jira Integration

## Important Git Commands

| Command | Purpose |
|----------|----------|
| git clone | Repository download |
| git fetch | Latest changes lana |
| git pull | Fetch + Merge |
| git merge | Branch combine karna |
| git rebase | Linear history banana |
| git revert | Safe rollback |
| git reset | Previous state par jana |

## Best Practices
- Direct main branch push disable rakho.
- Branch Protection enable rakho.
- SSO use karo.
- Code Review mandatory rakho.

---

# Ticket 2: Setup Repositories

## Purpose
Project ka code kis tarah repositories me organize hoga ye decide karna.

## Mono Repository
- Sab applications ek hi repository me.
- Easy code sharing.
- Repository size bahut badi ho sakti hai.

## Micro Repository
- Har application ka alag repository.
- Independent deployment.
- Easy access control.

## Decision
Humne **Micro Repository** choose kiya.

Reason:
- Independent deployment
- Better ownership
- Easy maintenance

---

# Ticket 3: Authentication (SSO Setup)

## Authentication

Authentication ka matlab hai:

**"User kaun hai?"**

System pehle verify karta hai ki login karne wala user valid hai ya nahi.

---

## What is SSO?

SSO (Single Sign-On) ek authentication method hai jisme user **sirf ek baar login** karta hai aur uske baad multiple applications ko bina dobara login kiye access kar sakta hai.

Example:

Ek baar Azure AD ya Okta se login kiya,
uske baad GitHub, Jenkins, SonarQube sab access ho jayenge.

---

## SSO Flow

1. User GitHub/GitLab open karta hai.
2. User Identity Provider (Okta/Azure AD/Keycloak) par redirect hota hai.
3. User Username, Password aur MFA complete karta hai.
4. Identity Provider user ko verify karta hai.
5. Verification successful hone ke baad Token/SAML Assertion generate hota hai.
6. Token GitHub/GitLab ko bheja jata hai.
7. User automatically login ho jata hai.

---

## Benefits

- Single Login
- Better Security
- Centralized User Management
- MFA Support
- Easy User Onboarding
- Easy User Offboarding
- Audit Logs Maintain hote hain

---

## Best Practices

- Hamesha SSO + MFA use karo.
- Local user accounts avoid karo.
- Password policy strong rakho.

---

# Ticket 4: Authorization

## Authorization

Authorization ka matlab hai:

**"User kya kar sakta hai?"**

Authentication complete hone ke baad system permissions check karta hai.

---

## Roles

### Admin
- Full Access

### Lead
- Merge
- Review

### Reviewer
- Sirf Review

### Contributor
- Feature Branch Push

---

## Best Practices

- Least Privilege Principle follow karo.
- Main branch protected rakho.
- Direct push disable rakho.

---

# Authentication vs Authorization

| Authentication | Authorization |
|----------------|---------------|
| Who are you? | What can you do? |
| Login Verify | Permission Verify |
| Pehle hota hai | Authentication ke baad hota hai |

Example

Login karna → Authentication

Main branch me Merge karna → Authorization

---

# Ticket 5: Commit & PR Workflow

## Workflow

```
Feature Branch
      │
Create PR
      │
Code Review
      │
Jenkins Build
      │
Lead Approval
      │
Merge to Main
```

## Rules

- Feature Branch naming follow karo.
- Minimum 2 reviewer approvals.
- Jenkins build pass hona chahiye.
- Sirf Lead merge karega.

## Standards

- Proper Commit Message
- Proper PR Description
- Meaningful Review Comments

---

# Ticket 6: Notifications

## Purpose

Developers ko important code events ki notification dena.

## Notifications

- PR Created
- PR Updated
- PR Approved
- PR Merged
- Main Branch Updated

## Setup

- Slack Webhook
- Email Notification
- Stakeholders ko notification

## Best Practices

- Sirf important events notify karo.
- Notification spam avoid karo.

---

# Ticket 7: Pre-Commit Hook

## What is Pre-Commit Hook?

Commit hone se pehle automatically validation run hoti hai.

Agar validation fail ho jaye to commit nahi hota.

## Purpose

- JIRA ID verify karna
- Standard Commit Message maintain karna
- Invalid commit rokna

## Steps

- pre-commit install karo.
- Hook configure karo.
- Commit message validate karo.

## Benefits

- Better Code Quality
- Standard Commit Messages
- Better Traceability
- CI Failures kam hote hain.

---

# Interview Important Questions

- What is VCS?
- GitHub vs GitLab vs Bitbucket
- Mono Repo vs Micro Repo
- Authentication vs Authorization
- What is SSO?
- SSO Flow
- What is Webhook?
- Why Branch Protection?
- Git Fetch vs Pull
- Merge vs Rebase
- Why Pre-Commit Hook?
- PR Workflow
- Jenkins Status Check
- Least Privilege Principle
- Why Direct Push to Main is Disabled?
````
