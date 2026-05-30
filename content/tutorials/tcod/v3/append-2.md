# Appendix 2: Combat Effects

The base combat system from the tutorial resolves every hit in one step: `damage = max(0, attack - target.defense)`. That is enough to build a playable game. Most games layer more effects on top: some attacks land harder, defenders can dodge or parry, certain weapons cut through armor, fire burns differently than a sword blow.

This appendix explains the most common effects, what they represent, how they are calculated, and what tends to go wrong with them. None of them are required. A good roguelike can be built with the base formula plus one or two additions. Add effects one at a time and test each before moving on.

!!! info "Before reading"
    These effects extend `Fighter.melee_attack()` in `game/entities/components/fighter.py`. [Appendix 1](append-1.md) already mentions critical hits and hit chance separation (dodge, parry, block) briefly in its "Other Adjustments" section. This appendix develops those ideas in full.

There is a lot of material in this appendix. You can read it in three passes:

- [**Individual effects**](#individual-effects): Critical hits, misses, dodge, parry, block, armor penetration, resistances, and status effects.
- [**Combining effects**](#combining-effects): Resolution order, independent rolls, and multiple attackers.
- [**Balance guidance**](#balance-guidance): Terminology, common problems, and where to start.

---

## Individual effects

### 1. Critical hit

A critical hit represents a strike that lands especially well: a precise blow to a weak spot, a lucky angle, or a moment of perfect timing. It deals more damage not because the attacker is stronger, but because this particular hit was more effective.

The most common approach is a probability check on the attacker's side, followed by a multiplier on the damage:

```python
if random() < attacker.critical_chance:
    damage = base_damage * attacker.critical_multiplier

else:
    damage = base_damage
```

A typical setup might be `critical_chance = 0.1` (10%) and `critical_multiplier = 2.0` (double damage). That means one in ten hits deals twice the normal damage, enough to feel meaningful without dominating every fight.

#### Variants

- **Attribute-dependent**: critical chance scales with dexterity, luck, or weapon skill. A specialized fighter crits more often than a generalist.
- **Guaranteed criticals**: some situations skip the probability check entirely. Common examples are attacking from behind (backstab), hitting a stunned enemy, or triggering a specific ability.
- **Criticals as effect triggers**: a critical hit can fire a secondary effect tied to a specific weapon or ability. A mace might have a small chance to stun on a critical; a poisoned blade might apply its venom with greater probability. This is a natural fit for weapon identity. One thing to watch: stacking extra damage and turn control into the same trigger tends to break balance faster than either effect alone.

#### Design note

Keep both values in check. A `critical_chance` above 0.25 or a `critical_multiplier` above 3.0 makes individual fights feel like coin flips. One unlucky string of enemy criticals can kill a player who believed they were safe.

The other side of this is that criticals are exciting precisely because they are **rare**. If every hit crits, nothing feels special.

---

### 2. Miss and accuracy / evasion

An attack can fail not because the defender did anything, but because the attacker simply missed. The sword swung wide, the arrow clipped the wall.

```python
if random() < attacker.miss_chance:
    damage = 0  # miss
else:
    damage = base_damage
```

Many games split miss into two separate stats: `attacker.accuracy` and `defender.evasion`. The outcome is the same (the attack lands or it does not), but two stats give independent control over each side of the equation.

```python
total = attacker.accuracy + defender.evasion
hit_chance = attacker.accuracy / total if total > 0 else 1.0
hit_chance = clamp(hit_chance, min_hit_chance, max_hit_chance)

if random() > hit_chance:
    damage = 0  # miss
else:
    damage = base_damage
```

---

### 3. Dodge

Dodging means the defender actively avoids the attack. They step aside, duck, or move out of the path of the blow.

The difference from miss matters mostly in context. When an attacker misses, something went wrong on their end. When a defender dodges, they did something right. Players notice this; it feels different to read "you dodge!" versus "the goblin misses you".

Dodge is usually calculated on the defender's side:

```python
if random() < defender.dodge_chance:
    damage = 0  # dodged
else:
    damage = base_damage
```

The attacker's accuracy appears here too, now as a penalty to the dodge chance rather than as part of a hit probability.

```python
dodge_chance = defender.evasion - attacker.accuracy
dodge_chance = clamp(dodge_chance, min_dodge_chance, max_dodge_chance)

if random() < dodge_chance:
    damage = 0  # dodged
else:
    damage = base_damage
```

The defender's evasion sets the base chance, and the attacker's accuracy can reduce it. Clamping keeps the probability in a reasonable range.

Evasion commonly scales with dexterity, agility or speed (even with luck if your game uses it). It can also depend on encumbrance: a character in heavy armor has a lower dodge chance than one in light armor. Small enemies may simply be harder to hit by virtue of their size.

!!! warning "Cap it"
    A dodge chance above 50% starts producing characters who are frustrating to fight. Something in the 5% to 30% range is enough to feel meaningful without breaking combat flow.

#### Miss vs dodge

Both result in no damage, but the cause is different. In a miss, the attacker failed to connect. In a dodge, the defender acted to avoid the blow. The distinction matters for feedback text ("The orc misses!" versus "You dodge the blow!") and for how the mechanic maps to character stats.

Some games collapse both into a single hit probability and skip the distinction entirely. That is a valid simplification.

---

### 4. Parry

A parry is an active defensive technique: the defender uses their weapon, shield, or skill to redirect the incoming attack rather than simply getting out of the way.

Unlike dodge, parry usually means interfering with the attack directly. That is why parry works better against melee attacks than ranged ones, and why it often requires holding a big weapon or shield.

```python
def melee_attack(self, target, engine, is_counterattack: bool = False):
    ...
    if not is_counterattack and random() < target.fighter.parry_chance:
        damage = 0
        target.fighter.can_counter_attack = True
    else:
        damage = base_damage
```

The `is_counterattack` parameter tells `melee_attack` whether this strike is itself a counterattack. If it is, the parry check is skipped entirely, breaking the recursion chain. The `can_counter_attack` flag is then free to mean exactly one thing: the defender has earned the right to strike back.

#### Design note: counterattack

If a successful parry grants a counterattack opportunity, parry becomes a defensive mechanic with offensive implications. That is a strong ability. A character who can parry every incoming attack and respond with a guaranteed hit (possibly with higher critical chance) becomes very difficult to fight.

In the tutorial's architecture, the natural place to resolve the counterattack is in `MeleeAction.perform()`, right after the attacker's strike resolves:

```python
entity.fighter.melee_attack(target, engine)

if target.fighter.can_counter_attack:
    target.fighter.can_counter_attack = False
    target.fighter.melee_attack(entity, engine, is_counterattack=True)
```

The flag is cleared before executing the counterattack. Passing `is_counterattack=True` ensures the counterattack cannot itself trigger a parry and another counterattack.

If you want the counterattack to have a higher critical chance, raise `critical_chance` temporarily before the call and restore it after.

Limit parry if it grants counterattacks. Reasonable restrictions: one parry per turn, only against frontal melee attacks, only when holding a weapon, or a stamina cost per use.

---

### 5. Block

Blocking means taking the hit but reducing its force. A shield absorbs part of the blow, a large weapon deflects some of the energy, or the character braces and accepts less damage.

Unlike parry, block does not need to represent a precise, skillful response. That makes it easier to justify against multiple attackers or attacks from unexpected directions.

- **Probabilistic block with percentage reduction**:

    ```python
    if random() < defender.block_chance:
        damage = base_damage * (1.0 - defender.block_reduction)
    else:
        damage = base_damage
    ```

- **Fixed block value** (always active, no probability check):

    ```python
    damage = max(0, base_damage - defender.block_value)
    ```

    Every hit that lands loses a fixed amount. Simple and predictable.

- **Mixed approach**:

    ```python
    if random() < defender.block_chance:
        damage = max(0, base_damage - defender.block_value)
        damage = damage * (1.0 - defender.block_reduction)
    else:
        damage = base_damage
    ```

    Block is a natural fit for shield mechanics. It can degrade over time (each block costs durability or stamina), which naturally limits it without adding hard caps.

#### Dodge vs parry vs block

All three prevent or reduce damage, but through different mechanisms:

- **Dodge**: the defender avoids the attack entirely. No contact.
- **Parry**: the defender redirects the attack. Contact happens, but it is neutralized.
- **Block**: the defender absorbs the impact. Contact happens and damage is reduced, not zeroed.

The practical difference shows up in design constraints: dodge works at any range and from any direction, parry typically requires a weapon and a frontal stance, and block usually depends on equipment. Each occupies a different design space.

---

### 6. Armor penetration

Armor penetration reduces how much of the defender's defense actually applies. A weapon with high penetration cuts through armor more effectively, dealing more damage against the same target than a normal attack would.

- **Flat penetration** subtracts a fixed amount from defense before calculating damage:

    ```python
    effective_defense = max(0, defender.defense - attacker.armor_penetration)
    ```

- **Percentage penetration** reduces defense by a ratio:

    ```python
    effective_defense = defender.defense * (1.0 - attacker.armor_penetration_ratio)
    ```

- **Mixed approach**:

    ```python
    effective_defense = defender.defense * (1.0 - attacker.armor_penetration_ratio)
    effective_defense = max(0, effective_defense - attacker.armor_penetration)
    ```

In all cases, `effective_defense` replaces `defender.defense` in the damage formula. Whether you use the linear formula from the tutorial or a smoothed one from Appendix 1, the substitution is the same.

Armor penetration lets you create meaningful differences between weapons, abilities, and classes. A mage's spell might ignore physical armor entirely. A rogue's dagger might have high penetration while a war hammer has none.

---

### 7. Damage resistance

Resistances reduce damage after it has been calculated, based on the type of damage. Fire, ice, poison, and physical are common categories, each with its own resistance value per defender.

```python
final_damage = base_damage * (1.0 - defender.fire_resistance)
```

A `fire_resistance` of `0.5` means the defender takes half fire damage. A value of `1.0` means full immunity. A negative value means vulnerability:

```python
fire_resistance = -0.25
final_damage = base_damage * (1.0 - fire_resistance)  # 25% more damage taken
```

Resistance applies after the base damage formula. The order matters: applying resistance before a critical multiplier means the critical amplifies already-reduced damage. Applying it after means resistances can reduce even amplified crits. Neither order is wrong, but pick one and be consistent.

Damage types create tactical variety. A fire-resistant troll does not simply have more HP; it requires a different approach. Keep resistance values finite and legible. Full immunities are fine when they are rare and clearly communicated.

---

### 8. Status effects

Some attacks do not just deal damage. They also inflict a condition: poison that drains HP over several turns, a stun that skips the target's next action, a slow that reduces movement range, a burn that ticks fire damage each round.

These are status effects, and they follow a common pattern:

```python
if attack_hit and random() < attacker.status_chance * (1.0 - defender.status_resistance):
    defender.apply_status(status_effect)
```

The application depends on whether the attack landed, the attacker's chance of inflicting the condition, and the defender's resistance to it.

Most status effects share four properties:

- **Application probability**: how likely the effect is to apply when the attack hits.
- **Duration**: how many turns the effect lasts.
- **Intensity**: how strong the effect is (how much damage per turn, how far movement is reduced).
- **Stack behavior**: does the same effect stack if applied twice? Does it reset the duration, or refresh it?

Implementing status effects well requires a component that ticks each turn, communicates its state to the UI, and expires cleanly. That is beyond the scope of this appendix. The pattern above is a reasonable starting point if you want to experiment.

---

## Combining effects

### 9. Attack resolution order

If you add several of these effects, the order in which you apply them matters. Different orders produce different results, and some orderings produce surprising math.

Whatever order you choose, keep it consistent. If the order differs between player attacks and enemy attacks, or changes between updates, you get unpredictable results that are difficult to debug.

#### Order changes the result

Applying the critical multiplier before defense means crits amplify raw attack before defense reduces it. That feels stronger and more impactful. Applying it after defense means crits amplify only the damage that survived the defense calculation. That feels more controlled.

Both approaches are valid.

---

### 10. Independent rolls vs single table

When several defensive effects are active at once (dodge, parry, block), there are two ways to resolve them: a series of independent rolls, or a single table roll.

**Independent rolls**:

```python
if random() < dodge_chance:
    result = "dodge"

elif random() < parry_chance:
    result = "parry"

elif random() < block_chance:
    result = "block"

else:
    result = "hit"
```

Easy to read and easy to write, but the probabilities are not what they appear. Each check only activates if all previous checks failed. The effective dodge chance is correct, but the effective parry chance is `parry_chance * (1 - dodge_chance)`, not `parry_chance`. The numbers on paper do not match what happens in play.

**Single table roll**:

```python
roll = random()

if roll < dodge_chance:
    result = "dodge"

elif roll < dodge_chance + parry_chance:
    result = "parry"

elif roll < dodge_chance + parry_chance + block_chance:
    result = "block"

else:
    result = "hit"
```

One roll, one outcome. The probabilities are exact as long as the total does not exceed 1.0. Easy to tune: you know exactly what fraction of attacks each outcome consumes.

Independent rolls are faster to prototype. Single table rolls are easier to balance. If your game has only one or two defensive effects, either works. With three or more, the single table approach will save you from probability surprises.

---

### 11. Multiple attackers

Most combat analysis focuses on a one-on-one fight. Roguelikes frequently put the player against several enemies at once.

This changes the math for defensive effects. A 60% chance to avoid one attack feels strong. Against three simultaneous attacks, the chance of avoiding all three is:

```python
chance_to_avoid_one_attack = 0.6
chance_to_avoid_all_three_attacks = 0.6 * 0.6 * 0.6  # = 0.216
```

That 60% individual probability becomes a 21.6% chance to avoid all three. The per-hit number looks fine; the cumulative result per turn does not.

#### How each defense behaves against multiple attackers

- **Dodge**: works as a passive probability against each attack individually, but cumulative failure rate climbs fast. It can also feel implausible if a character dodges five attacks from five different directions in a single turn. A per-turn penalty or cap keeps it believable.

- **Parry**: the most restrictive defense. It makes sense to limit parry to frontal melee attacks, attacks from visible enemies, or a maximum count per turn. If parry grants a counterattack (or raises the chance of one), limiting it against multiple attackers is especially important.

- **Block**: the most natural defense against several attackers. A shield absorbs part of each blow without requiring the precise timing that parry implies. It can degrade with each block (losing durability or stamina), adding a natural limit.

- **Miss**: no special handling needed. A miss is the attacker's failure, not a finite defensive resource.

#### Simple rules for limiting per-turn use

```python
if defender.parries_this_turn < defender.max_parries_per_turn:
    ...

if defender.blocks_this_turn < defender.max_blocks_per_turn:
    ...

dodge_chance = base_dodge_chance - attacks_received_this_turn * dodge_penalty_per_attack
...

block_reduction = max(0, base_block_reduction - blocks_this_turn * block_penalty_per_block)
...
```

Reset `parries_this_turn` and `blocks_this_turn` to `0` at the start of each turn, alongside `can_counter_attack`, in `handle_enemy_turns()`.

Or use a resource that depletes:

```python
if defender.stamina >= parry_stamina_cost:
    defender.stamina -= parry_stamina_cost
    # parry succeeds

else:
    pass  # cannot parry
```

!!! warning "Design rule"
    Active defenses that fully negate damage are much more dangerous to balance when they can trigger many times per turn. Block with partial reduction is safer than parry with full negation because partial reduction still allows some damage through.

For a simple roguelike: make dodge a passive probability, block a partial reduction, and parry a limited or special mechanic. That arrangement scales reasonably across one-on-one and multi-enemy situations.

> Against one enemy, evaluate dodge, parry, or block by its individual probability.
>
> Against several enemies, evaluate its cumulative effect per turn.

---

## Balance guidance

### 12. Terms to keep straight

These concepts are often confused, especially across different games and forums. For internal consistency, pick one meaning per term and use it everywhere.

| Term | Who causes it | What happens |
| --- | --- | --- |
| **Critical hit** | Attacker | Attack lands with extra force. Bonus damage. |
| **Armor penetration** | Attacker | Effective defense is reduced before damage calculation. |
| **Miss** | Attacker | Attack fails to connect. Defender did nothing special. |
| **Dodge** | Defender | Defender actively avoids the attack. No damage. |
| **Parry** | Defender | Defender redirects the attack. No damage, possible counterattack. |
| **Block** | Defender | Defender absorbs part of the impact. Reduced damage. |
| **Resistance** | Defender | Damage lands but is reduced by damage type. |
| **Immunity** | Defender | Damage or effect does not apply at all. |

---

### 13. Common balance problems

**Avoidance too high**: Dodge, parry, or block probabilities above 40-50% start producing characters who feel unkillable. Always clamp probabilities and test against multiple attackers, not just one.

**Stacking defenses**: A character with high dodge, high parry, and high block may combine them into near-immunity even if each individual value seems reasonable. Check the combined result.

**Explosive criticals**: A `critical_chance` of 0.25 combined with a `critical_multiplier` of 3.0 produces frequent one-shots. Keep the multiplier modest, or keep the chance low. Not both high at once.

**Full-negation block**: A block that reduces damage to zero on every trigger functions as free invincibility if it applies to every hit. Reserve full negation for rare or costly mechanics.

**Status effects that lock out play**: A stun lasting five turns removes the player from the game for five turns. Keep crowd-control durations short, or give the player a way to break out early.

**Asymmetric frustration**: A mechanic that feels great when the player uses it often feels terrible when enemies use it against the player. Crits, stuns, and high dodge are especially prone to this. Either limit how often enemies access powerful effects, or give the player meaningful counterplay.

**Missing feedback**: A player who takes damage without knowing why cannot learn or adapt. Always communicate what happened: `miss`, `dodge`, `parry`, `block`, `critical hit`, `resisted`, `immune`. A short message in the log is enough.

---

### 14. Summary

| Effect | Triggered by | Typical result | Balance risk |
| --- | --- | --- | --- |
| Critical hit | Attacker | Bonus damage | Too explosive if chance or multiplier too high |
| Miss | Attacker | No damage | Frustrating if too frequent |
| Dodge | Defender | No damage | Near-immunity if uncapped |
| Parry | Defender | No damage, counterattack option | Unbeatable if unlimited |
| Block | Defender | Reduced damage | Free invincibility if full-negate, unlimited |
| Armor penetration | Attacker | Defense partially ignored | Power creep if unchecked |
| Resistance | Defender | Damage reduced by type | Immune enemies removing entire damage sources |
| Status effect | Attacker | Ongoing condition | Play denial if duration too long |

#### Where to start

The tutorial's combat is functional with just `damage = max(0, attack - target.defense)`. If you want to extend it, a good first step is adding a critical hit: one attribute, one multiplier, one probability check. It adds surprise without touching the defense side of the equation.

After that, consider dodge or block before parry. Both are simpler to balance than parry because neither grants offensive options. Save parry for a later addition, once the rest of the system feels stable.

Damage types and status effects compound with everything else. Introduce them last, test them carefully, and expect to rebalance numbers you thought were settled.
