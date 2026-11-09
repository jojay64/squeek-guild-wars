# GuildWars contract (v1) - what is actually implemented

`src/GuildWars.sol` is the registry contract. It is the source of truth; where `SPEC.md` differs, this file and the contract win.

## Safety properties

- Never holds SQK, ETH or NFTs. No payable function, no `receive`, no `fallback`, no token transfer, no approval.
- No owner, no admin, no pause, no proxy. All parameters are immutable (`mining`, `staking`, `minStakerPower`).
- The only external calls are two read-only `staticcall`s (gas capped at 100,000) to the official SqueekMining (`checkpointStatus()`) and SqueekStaking (`accountPower()`, `stakeOf()`). They never revert the caller: a failed, reverting or malformed read only means "no bonus".
- No unbounded loops. The only loop is the guild-name character check (at most 24 characters).

## Rules as implemented

- Guilds: 5 to 20 seats, names up to 24 characters (letters, digits, space, hyphen), invite code stored as a hash, one guild per wallet, 24 h cool-down after leaving (also blocks creating a guild), captain can kick, transfer, open or close recruiting and rotate the invite code. A member may claim captain after 14 days of captain inactivity. Anyone can remove a non-captain member after 14 days without a check-in. A captain with other members must transfer the role before leaving; when the last member leaves, the guild is closed.
- Check-in: once per UTC day. 10 points; x2 if SqueekMining `nextRound` is greater than the value stored at the wallet's previous check-in; x1.25 (rounded down) if the wallet's SQK staking power reaches `minStakerPower` after ignoring the still-fresh part of the stake. Possible values: 10, 12, 20 or 25.
- Staking freshness: when `stakeOf(...).freshUntilRound` is still ahead of the current round, the effective power is `sqkPower * (principal - freshPrincipal) / principal`. This resists staking right before a check-in. It is a design choice based on the field names of the official ABI; check it against the official documentation.
- Score: a guild's season score is its season point total divided by its member count, kept per season (members who leave take their season points with them). Member counts and the guild's tier are snapshotted lazily at the first activity of a season, so a season's score does not change after the season ends.
- Seasons: week number, Monday 00:00 UTC.
- Wars: after a season ended, the captain of either guild calls `submitWar(season, guildA, guildB)`. Both guilds must have had at least one check-in that season, must share the same tier snapshot and must not be settled yet. Winner +300 tier points, loser +0 (nobody loses points), tie +150 each. Tiers: Bronze 0, Silver 1,000, Gold 3,000, Legend 6,000.

## Differences from SPEC.md

- No "bye" function: the contract cannot verify that a guild had no opponent, so a bye would be free points. A guild without an opponent simply gets no war that week.
- Wars are submitted by a captain of one of the two guilds (not by anyone), so third parties cannot pair guilds against their will.
- No checkpoint mission: a contract cannot prove that a given call settled a round.

## Known limitations (be honest about them)

- Captains choose their opponent inside their tier. Two cooperating guilds can trade wins to farm tier points, and a strong guild can pick a weak opponent. Tier points are cosmetic and carry no monetary value.
- Many wallets can still be created. The average-based score, the seat cap, the daily limit, the leave cool-down and the freshness rule only make that unprofitable, not impossible.
- `minStakerPower` is a raw value from `accountPower().sqkPower`. Pick it at deploy time with the official unit in mind.

## Deployed instance

Ethereum Sepolia, `0x4d9c2fa10fb70100d4c2390212b8f8de286a1d30` (block 12089806). It is the version adapted by the IMD swarm: the constructor only rejects the zero address (no code-length check), and `_readRound` decodes the three returned words as `uint256` so a malformed read can never revert a check-in. Everything else is unchanged. The contract is immutable, so the points below can only be addressed by the website or by a v2.

## Local simulation (40 test wallets, 4 guilds, 5 weeks)

Run offline on a local chain with mock SqueekMining and SqueekStaking, against the deployed source. Nothing was sent to any public network. Guild profiles: 10 daily players, 8 casual players, 12 mixed players and a 1-member guild; a mining round settles on about 60% of days.

What worked: points (10, 12, 20, 25), streaks, wars, tier points and ties behaved as specified over several weeks. Gas: `checkIn` about 74k on average (171k worst case, first check-in), `join` about 85k, `createGuild` about 196k, `submitWar` about 90k.

Findings (design weaknesses, not security bugs; rewards are cosmetic):

1. **Tiny guilds win on average.** The 1-member guild finished first in 4 of 5 seasons, ahead of the 10-member diligent guild. There is no minimum number of active members.
2. **Kicking inflates the average.** A captain can remove the lowest scorers just before the season ends; their points leave with them. In the test the average rose from 14.8 to 20.8. A kicked wallet can rejoin immediately (the cool-down only applies after leaving).
3. **Guild names are not unique.** A second guild can reuse any name, so the website must show the guild ID next to the name.

Possible mitigations. Website (no contract change): show guild ID and member count next to every score, offer a "5 or more members" filter, label small-guild results. Possible v2 contract: minimum members for war eligibility, no kicks during the last 24 hours of a season (or kicked points stay), a cool-down after a kick, unique names.

## Tests

15 offline tests (a local chain with mock SqueekMining and SqueekStaking contracts) passed before this was committed: constructor checks, guild creation and joining, recruiting and code rotation, exact check-in points (10, 12, 20, 25), the freshness rule, failing external reads, leaving and the cool-down, kick and captain rules, inactivity removal, wars (eligibility, double settlement, ties, size versus average, tier mismatch), season rollover on Monday and the absence of payable entry points.
