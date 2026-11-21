# Delay per lane: Brightwave Electronics (SHP03)

Reliability target: **98% on time** (no more than 24 hours late) on every lane.
Source: `starter/delay_per_lane.sql` on `freight.bookings` joined to `freight.schedules`, October 2026.

| Lane | Trade lane | Shipments | Avg delay (h) | On time (%) | Target (%) | Meets target? |
|---|---|---|---|---|---|---|
| DXB → FRA | Middle East-Europe | 58 | 1.8 | 100.0 | 98 | Yes |
| PVG → ORD | Transpacific | 70 | 1.3 | 100.0 | 98 | Yes |
| SIN → LHR | Asia-Europe | 61 | 0.7 | 100.0 | 98 | Yes |

## Findings

- Lanes missing the target: none. All three air lanes were 100% on time, with average delays under 2 hours.
- Carriers serving these lanes: Meridian Air Cargo (MER), Skybridge Cargo (SKY), Aquila Air Freight (AQL) and Falcon Cargo (FAL, DXB → FRA and PVG → ORD only). Every carrier was 100% on time on every lane.
- Implication: reliability is not the problem today. The platform's job is to catch the rare late flight fast enough, which matches the 5-minute freshness target, and to do it within the $35 per month budget.