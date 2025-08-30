# AI Price Tracker - Product Overview

## Purpose
Multi-store price monitoring application that tracks product prices across different retailers and provides intelligent alerts to users based on their subscription tier.

## Core Features
- **Product Tracking**: Monitor prices across multiple stores for the same product
- **Subscription Tiers**: Free, Premium, and Premium Plus with different check frequencies and limits
- **Dynamic Queue System**: Intelligent priority-based scraping that respects user subscription levels
- **Price Alerts**: Automated notifications when target prices are reached
- **AI-Powered Extraction**: Uses DeepSeek API for intelligent price parsing from scraped content

## Business Model
- **Free Tier**: 5 products, daily checks, 7-day history
- **Premium Tier**: 25 products, 6-hour checks, 30-day history  
- **Premium Plus Tier**: 100 products, hourly checks, 90-day history, API access

## Key Differentiators
- Queue-based architecture that scales without modifying GitHub Actions
- Fair resource allocation ensuring all users get guaranteed check frequencies
- Tier-based data retention and processing priorities
- Cost-optimized using free tiers of modern services (Neon, Vercel, GitHub Actions)