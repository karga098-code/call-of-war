# Call of War: Grand Strategy MMO

**Massive Online Multiplayer Grand Strategy War Game**

A real-time, browser-based, fully persistent grand strategy game set in the 1500s with deep systems for logistics, trade, stealth warfare, and coalition politics.

## 🎮 Game Features

- **Real-time Persistent World** - 4x game speed, accelerating progression
- **Strategic Geography** - Province-based map system with cities and logistics
- **Military Complexity** - 60+ unit types across infantry, armor, air, and naval
- **Advanced Logistics** - Supply chains, resource distribution, trade networks
- **Diplomacy & Coalition** - Alliances, peace treaties, embargoes, reparations
- **Stealth Warfare** - Hidden units, radar systems, electronic warfare, scout detection
- **Economy** - Player-driven global market, resources, currency
- **Intelligence** - Fog of war, intel sharing, multi-layer vision systems
- **Research** - 8-10 tier technology tree with progression
- **Naval Transport** - Embarkation/disembarkation with vulnerability windows

## 🏗️ Architecture Overview

### Backend
- Node.js + Express API server
- WebSocket real-time communication
- PostgreSQL persistent database
- Redis for state caching & real-time updates
- Game loop engine (100ms tick)

### Frontend
- React + TypeScript
- WebGL map rendering
- Real-time WebSocket updates
- Responsive dashboard UI

### Infrastructure
- Docker containerization
- Kubernetes orchestration
- Load balancing
- Database replication & backup

## 📁 Project Structure

```
├── backend/              # Node.js game server
├── frontend/             # React web client
├── database/             # PostgreSQL schemas
├── docker/               # Container configuration
├── docs/                 # Technical documentation
├── config/               # Environment configs
└── scripts/              # Deployment & maintenance
```

## 🚀 Quick Start

### Prerequisites
- Node.js 18+
- PostgreSQL 14+
- Redis 6+
- Docker & Docker Compose

### Development

```bash
# Clone repository
git clone https://github.com/karga098-code/call-of-war.git
cd call-of-war

# Install dependencies
npm install

# Setup environment
cp .env.example .env

# Start with Docker Compose
docker-compose up

# Server runs on http://localhost:3000
# Client runs on http://localhost:3001
```

## 📊 System Documentation

- [Map System](docs/SYSTEMS.md#map-system)
- [Combat System](docs/SYSTEMS.md#combat-system)
- [Logistics System](docs/SYSTEMS.md#logistics-system)
- [Diplomacy System](docs/SYSTEMS.md#diplomacy-system)
- [API Reference](docs/API.md)
- [Database Schema](docs/DATABASE.md)
- [Deployment Guide](docs/DEPLOYMENT.md)

## 🎯 Game Balance

- Real-time movement, combat, and economy
- 4x game speed with daily acceleration
- Resource production scales with population
- Army upkeep based on manpower (conscription system)
- Volunteer recruitment for manpower generation

## 📱 Supported Browsers

- Chrome 90+
- Firefox 88+
- Safari 14+
- Edge 90+

## 📄 License

MIT License - See LICENSE file

## 🤝 Contributing

See CONTRIBUTING.md for guidelines

## 📧 Contact

Project maintained by karga098-code
