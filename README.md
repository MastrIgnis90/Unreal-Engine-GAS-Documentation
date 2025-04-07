# Ability System Component
Connects GAS to the games actors.

Manages Gameplay Abilities, Attribute Sets, Gameplay Effects, and Gameplay Cues for an actor.

---
# Gameplay ability
*What can this actor do within GAS? How do they do this?*

Defines the capabilities of actors within GAS's context.

---
# Attribute Set
*Who is this actor within GAS? What qualities do they have?*

Defines the 'Attributes' of actors within GAS's context.

---
# Gameplay Effect
*How can GAS affect these actors?*

The Gameplay effect life cycle is made up of 4 separate parts
1. Gameplay Effect class
2. Gameplay Effect Spec struct
3. Active Gameplay Effect struct
4. Active Gameplay Effect Handle struct

## Gameplay Effect FAQ

**You have just attempted to apply a gameplay effect. How do I know if it succeeded?**  
The function that attempts to apply a gameplay effect will return an ActiveGameplayEffectHandle. Run `ActiveGameplayEffectHandle::WasSuccessfullyApplied()` on it.

**When should I create a GameplayEffectComponent vs a GameplayEffectExecution?**
GameplayEffectComponents should be made when the functionality relates to how the GameplayEffect itself works.  
eg, Should this effect be applied? This effect does X in addition to base functionality. 

GameplayEffectExecutions should be made when a GameplayEffect needs some more complex functionality or needs to run logic outside of GAS
eg, GameplayEffect deals damage to an enemy but needs to reduce the damage it does by the target's defense before executing. [^1]

[^1] This same effect can and should be handled by the attribute class' Pre/PostGameplayEffectExecute functions. The GameplayEffectExecution implementation should be reserved for edge cases while the AttributeSet implementation should be used for more general cases.

## Gameplay Effect class
**Overview**  
- Typically a data-only BP class.
- Const (Not editable at runtime)
- **not** instanced!

Gameplay Effect class contains the core functionality of what this Gameplay Effect actually does.  
It achives this through 3 different parts:  
1. Gameplay Effect Component
	- Describes what this gameplay effect can do
2. Gameplay Effect Modifier
	- Describes what attributes this gameplay effect will effect 
3. Gameplay Effect Execution
	- Describes any other custom logic this Gameplay Effect can run

## Gameplay Effect Spec struct
**Overview**  
- Generated at runtime
- Can be modified before submission to ASC
- Instanced!

Stores all dynamic information for a Gameplay Effect before submission to an ASC.  
This includes:  
1. SetByMagnitude values
2. Stack count
3. Level
4. **Effect Context**
	- Instigator (The actor that owns the ASC that produced this effect)
	- Effect Causer (The actor that actually applied the effect)
	- Source Object (The object which created this effect) ***Check this again! This could potentally be used to apply a gameplay effect from an actor without an ASC and/or for UObject instead of AActors!***
	- eg, Player has a sword which can create poisoned puddles. Instigator = Player, Effect Causer = Sword, Source Object = Poison Puddle.

## Active Gameplay Effect struct
**Overview**
- Generated at runtime
- Pooled
- replicated.

Container that binds an active Gameplay Effect on a character with its Gameplay Effect Spec struct. Each 1 is unique and only generated once.  
Data in this struct is not guarenteed to be the same. Data can be overwitten when a new effect comes in or an old effect expires. Thus this is effectively pooled.  
Do not rely on this data always staying relevant! 

1. Handle
	- Globally unique ID. This can be used to find owner of this gameplay effect without any other data. 
2. Spec
	- The submitted Gameplay Effect Spec struct
	
### Active Gameplay Effect Handle struct
**Overview**
- Not directly linked to Active Gameplay Effect.
- Tracks if this gameplay effect was activated or not. 

Container that stores the unique ID of the represented ActiveGameplayEffect and a bool to tell if this effect was executed or not.

**NOTE:** The stored ID of the handle is only used to track ActiveGameplayEffects that have a duration! (Has Duration **or** Inifinite). Any GameplayEffects that are set as Instant will still create a one of these handles but the ID this handle stores will be invalid!

*id* -> Uses GlobalActiveGameplayEffectHandles::Map static property to translate stored ID into represented ActiveGameplayEffect. If this is not `INDEX_NONE` then the *bool* can never be false. 

*bool* -> False by default, set to True if the GameplayEffect actually executes on target. Execution can be blocked by GameplayEffectComponents like `Require Tags to Apply/Continue This Effect`

**Examples**

1. *id* = `INDEX_NONE`. *bool* = False
	- Default state when running blank constructor (`FActiveGameplayEffectHandle()`)
	- *id* being `INDEX_NONE` means that this handle was either never initialized properly, was used on an Instant GameplayEffect, or was valid at some point but the duration expired and this handle was invalidated. 
	- *bool* being False means that this handle wasn't initialized propertly, or that the GameplayEffect related to this handle never passed it's application requirement.  
	
**Verdict:** This Handle never had any impact on the game. It was either never used properly, or the GameplayEffect related to this handle never passed it's application requirement and failed.
	
	
2. *id* = Not `INDEX_NONE`. *bool* = True
	- *id* being valid means that this handle is tracking an ActiveGameplayEffect. 
	- *bool* being true means that this has had some level of effect on the game.  
	
**Verdict:** This handle is tracking an ActiveGameplayEffect with a duration and is actively effecting the game. 

3. *id* = `INDEX_NONE`. *bool* = True
	- *id* being invalid but the *bool* being true means that this was used on an Instant type GameplayEffect.
	- *bool* being true means that this effect did get the chance to activate.  
	
**Verdict:** This handle was used to apply an Instant type GameplayEffect which did effect the game but is not anymore. 

4. *id* = Not `INDEX_NONE`. *bool* = False  

**Verdict:** This is an impossible state for this struct. If you come across this then something is very wrong. 

---
# Gameplay Cue


Gameplay Effect system is made up of 3 different parts
	- Gameplay Effect class
		- Pointer to 
	- Gameplay Effect Spec struct
	- Active Gameplay Effect
