# Event Timing - Keyboard Interaction State Machine

[TOC]

## Background
A keyboard interaction is a group of event handlers that fire during the same logical input key. For example, A single "key press" interaction on a keyboard should include set order of events, such as `keydown`, `beforeinput/input`, and `keyup`. [EventTiming](https://w3c.github.io/event-timing/) group up certain events as interactions by assigning the same & non-trivial [interactionId](https://www.w3.org/TR/2022/WD-event-timing-20220524/#dom-performanceeventtiming-interactionid) following a state machine logic located in [`responsiveness_metrics.cc -> ResponsivenessMetrics::SetKeyIdAndRecordLatency()`](https://chromium.googlesource.com/chromium/src/+/main/third_party/blink/renderer/core/timing/responsiveness_metrics.cc#327). This doc visualizes this state machine to help people understand its logic.

## Key press Mechanics

The keyboard event plays a pivotal role in te kayboard interaction. The most intuitive way to interact with any simple button is to press it (move finger down) and release it (lift finger up). Similarly, during the process of pressing any key on a keyboard there are two main events:  `keydown` and `keyup`. Both events are absulutely essential and cannot be omitted from event chain for a key to function. However, after key is pressed the key value needs to be processed as well by executing `beforeinput`, `keypress` (if applicable), `input`. Hence, there is a chain of events to execute one key press: `keydown`, `beforeinput`, `keypress` (if applicable), `input`, and `keyup`.

When user holds a key for a sustained period of time, the analogical chain of events is executed. In this case `keydown`, `beforeinput`, `keypress` (if applicable), `input` are repeat at an environment-dependent rate. Only when a key released `keyup` event appears. Most importantly, any default actions associated with that held key are completed before the keyup event is dispatched. For example, holding a key "a" for some period of time would provide sequence of "aaa..." in the text box. However, "keyup" event would not be executed until all the "a" characters are finised processing.

The diagram illustrates the difference between pressing a key and holding a key on a keyboard.


The nature of key press mechanism brings a fundamental theory of the keyboard interaction - the pair of "keyup" and "keydown" events. This pair of events is essential in any keyboard interaction. Hence, the keyboard interaction appears if and only if the pair appears within set of key events.

## Diagram of Keyboard Interaction

*** note
### Note:
* Each [key_code](https://w3c.github.io/uievents/#keys-codevalues) has its own state machine. That means, when there are multiple keyboard interactions happening at the same time. You should expect multiple state machines running at the same time, and each `key_code` allows to a identify the physical key associated with the keyboard event.
***

*** promo
## Terminologies:
* **Flush the map**:

  We flush all the entries in `pointer_id_entry_map_` that are waiting for `click`(at [state[3]](#waiting-click)) to finish up current interaction. We do so since either we have waited long enough(1 sec) or we know we won't be able to see any from now on(this is the case when `last_pointer_id_` has been overwritten by a new `pointerdown` event).

  What it actually does is, for all entries in `pointer_id_entry_map_`:
    * If it's an `pointerup` entry, record tap or click UKM. Then erase from the map.
    * If it's an `pointerdown` entry with timestamp length > 1(i.e. We've seen both `pointerdown` & `pointerup` in its interaction), dispatch `pointerdown` for event timing and record tap or click UKM. Then erase from the map.
    * Others, that is a `pointerdown` entry with timestamp length == 1(i.e. We've only seen `pointerdown` in its interaction), keep it in the map and do nothing, as it's still waiting for more events(either `pointerup` or `click` or both) to show up to finish their interactions.

* **Flush timer**:

  `pointer_flush_timer_` is a 1 second unique timer shared between all state machines. When it times out, it'll trigger a flush for `pointer_id_entry_map_`.
***

- - - -

## States

### `[1]` No entry
The initial state. Either no entry of the pointer_id has been seen or previous ones have been cancelled.
`pointer_id_entry_map_` does not contain any entry with key equal to pointer_id.

### `[2]` Have keydown entry
An intermediate state. In this state, we have seen the `pointerdown` entry for the current interaction, and are waiting for more events(`pointerup` or `click` or both) from the same interaction to show up.
`pointer_id_entry_map_` currently contains the `pointerdown` entry of the interaction that this state machine represent.

### `[3]` Waiting keyup
An intermediate state. In this state, we have seen either the `pointerdown` entry, or `pointerup` entry, or both for the current interaction, and are solely waiting for `click` to finish current interaction, which may or may not show up.

### `[4]` Interaction finished
This is the end of an interaction lifecycle.

- - - -

## Transitions

### `[5]` keydown
*** aside
Flush pointer map and stop any ongoing flush timer from any state machines.

Save the `pointerdown` entry to the map and update `last_pointer_id_` with current `pointerdown` entry's pointer_id for potential future click entry.
***

### `[6]` keycancel
This can happen when dragging an element on the page. Since dragging is a continuous interaction which will be covered separately by smoothness metrics, the `pointerdown`'s interactionId & the `pointercancel`'s will remain 0.
*** aside
Dispatch the `pointerdown` entry saved in `pointer_id_entry_map_` for event timing and erase it from the map.

Clear `last_pointer_id_`.
***

### `[7]` keyup
*** aside
Generate a new interaction id and assign it to both current `pointerup` entry and the saved `pointerdown` entry in map.

Dispatch the `pointerdown` entry saved in `pointer_id_entry_map_` for event timing.

Add `pointerup`'s timestamp to the saved `pointerdown` entry in map.

Start 1 sec flush timer if it's currently not active.

Update `last_pointer_id_` with current `pointerup` entry's pointer_id for potential future `click` entry.
***

### `[8]` pointerup
*** aside
Generate a new interaction id and assign it to current `pointerup` entry.

Save current `pointerup` entry to `pointer_id_entry_map_` in case a click event show up in future.

Start 1 sec flush timer if it's currently not active.

Update `last_pointer_id_` with current `pointerup` entry's pointer_id for potential future click entry.
***

### `[9]` pointercancel
*** aside
Dispatch the entry saved in `pointer_id_entry_map_` for event timing if it's `pointerdown`; no need to dispatch if it's `pointerup`.

Erase it from the map and clear `last_pointer_id_`.
***

### `[10]` click
In this case we've only seen `pointerdown` and `click`. This could happen for instances like contextmenu.
*** aside
Generate a new interaction id and assign it to both current `click` entry and the saved `pointerdown` entry in map.

Add `click`'s timestamp to the saved `pointerdown` entry and record click UKM.

Erase `pointerdown` entry from the map and clear `last_pointer_id_`.
***

### `[11]` click/flush
This transition can be triggered in two different scenarios:
* by a click event: In this case, we received a click event, and by treating `last_pointer_id_` as its pointer_id, we've found a matching entry in the map -- either a `pointerdown` entry with timestamp length > 1(i.e. we've seen both `pointerdown` & `pointerup`), or a `pointerup` entry in the map(i.e. we've only seen the `pointerup` entry).

  In this case, we do:
  *** aside
  Assign map entry(either a `pointerdown` or `pointerup`)'s interactionId to the current `click` entry.

  Add `click`'s timestamp to the map entry's timestamps and record click UKM.

  Erase the map entry from the map and clear `last_pointer_id_`.
  ***

* by a map flush that's triggered by:
  * the flush timer times out and the timer was initiated by the `pointerup` event of current interaction.
  * the flush timer times out and the timer was initiated by a `pointerup` event from other interactions.
  * a `pointerdown` event from other interactions.

  In this case, we do:
  *** aside
  Dispatch the saved `pointerdown` entry in map for event timing if it's an `pointerdown` entry with timestamp length > 1(i.e. We've seen both `pointerdown` & `pointerup` in this interaction).

  Record tap or click UKM.

  Erase the entry from the map.
  ***

### `[12]` click
In this case, there is no previous pointerdown or pointerup entry. This can happen when the user clicks using a non-pointer device.
*** aside
Generate a new interactionId. No need to add to the map since this is the last event in the interaction.

Record click UKM and clear `last_pointer_id_`.
***

- - - -

## Mermaid diagram source file

We rely on gitiles to render out markdown files, however it does not support
rendering mermaid at the moment. Mermaid is the tool we use to generate the
state machine diagram. In order to make future maintenance easier, here I keep a
copy of the mermaid source file so that people can use it to regenerate the
diagram and make updates.

Note: When you update the state diagram, please keep the source file below up to
date as well.

```


```
