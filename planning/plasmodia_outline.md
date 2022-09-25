################################################################################
Changes to try:

Two types of tiles
- Producer
- Consumer

Consumers need to be chained?
################################################################################

Name: Plasmodia (Expansion? Seizedom? Domain? Dominion? Extraction? Hexation?)

Goal is to expand territory. Can take unoccupied territory or territory from others.

Setup:
    - Reset Map
        Reload blank_map.png
        Pick starting tiles for each player (1 tile at level 1)
    - Reset Tiles sheet
        Clear columns C, D, E, F, H, and I (Influence for P1 & P2, Owner, Tile Level, Nearest Hub Steps & Level)
        Mark the starting tiles for each player. Level 1 Hub starts at 200 Influence
    - Reset Game Info sheet
        Energy stockpile set to 300
        (Other cells are functions)

Prototype Play Steps:
    1) Player decides on actions. Only one action per initiating tile.
    2) Execute actions (Operate on own tiles before unclaimed tiles before enemey tiles)
        - Pay the Energy cost (Game Info sheet)
        Level up (Cost (level + 1) * 100E):
            Update Influence level and Tile Level cells for the target tile.
            If increasing from levels 0->1, clear the Nearest Hub cells for this tile to avoid confusion later.
            Must manually update the Nearest Hub cells for other nearby tiles.
        Abandon:
            Calculate Energy recovered from cell's influence level
            Clear the Influence level for the player, Owner, and Tile Level cells for the target tile.
            If level > 0, must manually update the Nearest Hub cells for other tiles which used this hub.
        Spread:
            If unclaimed
                Add to Influence cell for the target tile.
                If influence passes threshold (100 I):
                    Player acquires tile. Update Owner and Nearest Hub cells for tile.
            If own tile:
                Add to Influence cell for the target tile.
            If enemey tile (attacking):
                Subtract from Influence level cell for other player
                If enemeny influence <= 0:
                    Update Influence, Owner, Tile Level, and Nearest Hub cells according to section below.
                    (i.e. attacker bonuses, ownership changes, level reduction, hub dependencies)
    3) Game Logic
        Add Tile Production Sums (cells O2 & P2) to the respective Energy Stockpile on Game Info sheet.
        Apply Influence Exerted:
            For now, let players decide how it's distributed, but must go only to adjacent tiles
            Note, can be applied toward enemy tile too, but there are no bonuses if the tile flips.
            Note, because it would be too difficult to track new tiles vs old tiles, newly acquired also exert influence. (In the real version they would have to wait a turn)

One resource:
    Energy which is harvested from hubs (automatically)

Map:
    For now just uniform
        Every tile potentially provides 5 energy per turn
    Maybe in future have randomly distributed bonuses or distribution of rates

Tiles:
    Hex tiles
        Production Rate:
            Actual production rate depends on tile level and transport efficiency to a hub.
            E = baseline_rate * (1 - level/5) * transport_fu()
            baseline_rate = 5 E/turn            <- for now with a uniform initial map
            transport_fu = {
                            if level > 0 : 1              <- i.e. exports through itself
                            if level == 0 : max_of_any_tile_within_5_steps( (level_other - steps_to_other + 1) / 5)
                    }

            functional results:
                - tiles can never produce more than their level allows (1-level/5)
                - tiles at level zero need access to a hub or else will not produce, but any level higher can export itself at maximum transport efficiency.
                - tiles can export through a neighboring hub, but final production rate further reduced depending on distance to and level of the hub.
                - first step is free (ie no distance penalty for immediately adjacent hubs)
                - e.g. max rate acheived from a level zero tile adjacent to a level 5 tile.
                - e.g. Players must balance spacing out hubs for economic gain against leveling up tiles for defense.

        Stored Influence
            Starts at 0 for each player, can go up or down separately
            If tile unclaimed, Influence reduces cost of to acquire tile
            If tile unclaimed and influence for a player reaches a threshold (100 I), player automatically acquires tile for free
            If tile claimed, only owner's Influence goes up or down (other's are frozen)
                Up through player's actions or tile adjacency influence
                Down through enemy's actions or tile adjacency influence
            If tile claimed and owner's influence <= 0, then owner loses control of tile and influence from other players unfrozen

        Hub levels:
            Levels are zero to five
            Higher level reduces tile's production rate but can increase rates of nearby tiles (see Production Rate section)
            Higher levels increase the influence exerted on adjacent tiles
                - In general, exerted = level * 2 + 2
                - If tile is unclaimed, adds to player's influence on tile, making them cheaper to acquire
                - If tile belongs to self, the two tiles compare exerted influence and the weaker tile gains the difference, if any
                - If tile belongs to enemy, the two tiles compare exerted influence and the weaker tile loses the difference, if any


Actions: (Only one action per initiating tile per turn. Resolved at end of turn)
    Spread (Variable cost)
        Only tiles within territory or immediately adjacent to own tiles.
        If unclaimed tile:
            Applies cost towards influence in the tile (1 E -> +1 I)
            If the stored influence reaches a threshold (100 E), then adds tile to territory.

        If own tile:
            Applies cost towards influence in the tile (1 E -> +1 I)
            i.e. increases "defense" or "repairs" from an attack

        If enemy tile:
            Reduces enemy’s influence in tile by attack cost (1 E -> -1 I)
            If enemy influence I <= 0,
                Defender loses control of tile. Influence from all players are unfrozen.
                Any enemeny influence below zero is transferred to attacker as positive influence (in addition to any existing influence)
                Attacker gets influence bonus = max(level-2, 0) * 50I + 25 I
                Attacker gets energy plunder = level * 10E + 25 E
                Level reduced to max(level-2, 0)
                Note, attacker may also acquire tile if their influence goes above threshold after adding attack residual and bonus

    Abandon (No cost)
        Give up tile and recover some energy from existing influence.
        Influence from all players are unfrozen
        Returns E = 80% of I if unprovoked, 40% of I if tile attacked prior turn
        Influence in tile from abandoning player set to -80% of their former influence level

    Level up (Cost (level + 1) * 100E)
        Increases level (Levels zero to five)
        Add influence to tile (1 E -> +1 I)
        Reduces production rate from tile by 20%
            (100% -> 80% -> 60% -> 40% -> 20% -> 0%)
        Increases influence exerted on adjacent tiles (which reduces cost to expand)

X Extraction mode:
X     Fixed supply of energy in untapped tile
X     Or extraction only from adjacent unclaimed tiles
X     Or finite  supply of quick harvest that declines to infinite slow harvest (ie rate starts high and decays to a baseline rate)

X Explore: ( only non uniform maps)
X     Can grow filaments with  tiles adjacent to existing territory or filaments
X     Reveals details about adjacent tiles
X     Adds negative influence to the tile, makes it more expensive to acquire later
