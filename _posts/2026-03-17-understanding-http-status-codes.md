---
title: "Understanding HTTP Status Codes"
categories:
  - Dev Notes
tags:
  - HTTP
  - Web
lang: en
date: 2026-03-17
excerpt: "A quick reference for the most common HTTP status codes and when they're used."
---

## 1xx — Informational

Rarely seen in practice. `100 Continue` tells the client to keep sending the request body.

## 2xx — Success

| Code | Meaning |
|------|---------|
| 200 | OK — request succeeded |
| 201 | Created — resource was created |
| 204 | No Content — success, but nothing to return |

## 3xx — Redirection

| Code | Meaning |
|------|---------|
| 301 | Moved Permanently |
| 302 | Found (temporary redirect) |
| 304 | Not Modified (use cached version) |

## 4xx — Client Error

| Code | Meaning |
|------|---------|
| 400 | Bad Request — malformed syntax |
| 401 | Unauthorized — authentication required |
| 403 | Forbidden — authenticated but no permission |
| 404 | Not Found |
| 429 | Too Many Requests — rate limited |

## 5xx — Server Error

| Code | Meaning |
|------|---------|
| 500 | Internal Server Error |
| 502 | Bad Gateway |
| 503 | Service Unavailable |

## Quick Tip

> If you're building an API, be specific with your status codes. Don't return `200` for everything — it makes debugging much harder.
