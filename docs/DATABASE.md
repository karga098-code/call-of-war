# Database Schema - Call of War

## Core Tables Structure

### users table
- id, email, username, password_hash, 2fa_enabled

### countries table
- id, player_id, name, capital_province_id, population, manpower

### provinces table
- id, name, x, y, terrain, owner_country_id, population

### cities table
- id, name, province_id, type, population, garrison_army_id

### armies table
- id, country_id, current_province_id, status, target_province_id

### units table
- id, army_id, unit_type, count, health, morale, experience

### resources table
- country_id, food, oil, steel, electronics, money

### diplomacy_deals table
- id, type, country_a_id, country_b_id, status, terms

### market_orders table
- id, seller_country_id, type, resource_type, quantity, price

### buildings table
- id, city_id, type, level, is_constructing, completion_time

### research table
- country_id, unit_type, tier, is_completed, completion_time

### coalitions table
- id, name, leader_country_id, created_at

### news_events table
- id, type, description, related_countries, created_at

