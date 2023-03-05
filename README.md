# Event Timing - Keyboard Interaction State Machine

## Background
A keyboard interaction is a group of event handlers that fire when pressing a key on a keyboard. For example, A single "key pressed" interaction should include set order of events, such as `keydown`, `beforeinput/input`, and `keyup`. [EventTiming](https://w3c.github.io/event-timing/) group up certain events as interactions by assigning the same & non-trivial [interactionId](https://www.w3.org/TR/2022/WD-event-timing-20220524/#dom-performanceeventtiming-interactionid) following a state machine logic located in [`responsiveness_metrics.cc -> ResponsivenessMetrics::SetKeyIdAndRecordLatency()`](https://chromium.googlesource.com/chromium/src/+/main/third_party/blink/renderer/core/timing/responsiveness_metrics.cc#327). This doc visualizes this state machine to help people understand its logic.

## Key press Mechanics

The keyboard event plays a pivotal role in the keyboard interaction. The most intuitive way to interact with any simple button is to press it (move finger down) and release it (lift finger up). Similarly, in the process of pressing any key on a keyboard there are two main events:  `keydown` and `keyup`. Both events are essential and cannot be omitted from the event chain. In addition, after a key is pressed the key value needs to be processed by executing `beforeinput`, `keypress` (if applicable), `input`. Hence, there is set order of events to execute one keyboard event - key press: `keydown`, `beforeinput`, `keypress` (if applicable), `input`, and `keyup`.

When user holds a key for a sustained period of time, the analogical chain of events is executed. In this case `keydown`, `beforeinput`, `keypress` (if applicable), `input` are repeat at an environment-dependent rate. Only when a key released the event `keyup` appears at the end of the events chain. Most importantly, the default actions appeared before the `keyup` gets the priority. They are completed before the keyup event is dispatched. For example, holding a key "a" for some period of time would provide sequence of "aaa..." in the text box and "keyup" event would not be executed until all the "a" characters are finised processing.

The diagrams illustrate the difference between pressing a key and holding a key on a keyboard.
Pressing a key           |  Holding a key
:------------------:|:-------------------------:
[![](https://mermaid.ink/img/pako:eNqNUktPwzAM_iuRj6ib1sfWJQcOaBeEgAM31mnKWo9VtGmVpoUy7b-Tvra2Q8Apjr-HHTtH8JMAgUGmuMJVyN8kjyeF5QlPiGSLQsmSMPKUkCb0xIEXuH3HMkg-hEYe2qgFdrhPJIYizRUjd5dLC7fAfXX0vFKJWVZ71dEFyNOmhD5HbAwahLS3qmFCyPpmQyaTW9L1XifPD6mQUf86ImHWuZAqW2sGtL5jo_G58DEiP1PHk9AS_8Al9xVKUvAoRyLRx7DQBX9x6LSr50eSp4HeT5_e9x9KBoo_BJfhr7tQb5rvIgw2g9Ya6F-Frkvk6XnQEiPk2fjher9XLfUte1uveHrLYECMMuZhoD_vsWJ6oA4YowdMhwHueR4pDzxx0lSeq-SlFD4wJXM0oOm5_e7A9jzKdDbl4jVJ4o6kr8CO8AmMWlOXzhyTzuaWQ92laUAJzFpMHUpN23Udas-WlnUy4KvWz6bUpu7CmlPLNucL116evgFfXSsx?type=png)](https://mermaid.live/edit#pako:eNqNUktPwzAM_iuRj6ib1sfWJQcOaBeEgAM31mnKWo9VtGmVpoUy7b-Tvra2Q8Apjr-HHTtH8JMAgUGmuMJVyN8kjyeF5QlPiGSLQsmSMPKUkCb0xIEXuH3HMkg-hEYe2qgFdrhPJIYizRUjd5dLC7fAfXX0vFKJWVZ71dEFyNOmhD5HbAwahLS3qmFCyPpmQyaTW9L1XifPD6mQUf86ImHWuZAqW2sGtL5jo_G58DEiP1PHk9AS_8Al9xVKUvAoRyLRx7DQBX9x6LSr50eSp4HeT5_e9x9KBoo_BJfhr7tQb5rvIgw2g9Ya6F-Frkvk6XnQEiPk2fjher9XLfUte1uveHrLYECMMuZhoD_vsWJ6oA4YowdMhwHueR4pDzxx0lSeq-SlFD4wJXM0oOm5_e7A9jzKdDbl4jVJ4o6kr8CO8AmMWlOXzhyTzuaWQ92laUAJzFpMHUpN23Udas-WlnUy4KvWz6bUpu7CmlPLNucL116evgFfXSsx)  |  [![](https://mermaid.ink/img/pako:eNqFU01v2zAM_SuEjkMSWF7zYQHroegORbHtsNvioGBspjEqS4Yse8uC_PfRX42Tot3JFN_jI59EH0ViUxJKlB493Wf47DCf1mFsYmPsExnvDqDgu4U2jM0ea3p6oUNqfxsGHvuoB7a0s44yU1Rewd35MOA98jDOsVjhqCxbsTY6A1XR9eDvFZvSDoHh1EwMAOtPG5hOb2EYvk2-OmmQKwdrFtladOnXmjkzRwWhhy_gXUUbYB5kJexJp9AbZb0LiXE31uN0giYh_Q71-p64JNmjw8STgxp1ReAooaxuPb2rMNTe__gGVZHy443pY_3LkouK_xScX2Y9hLwGuNWUbi5G66APGo1Ge9ujMfYRXhU8AGoNKe2w0h6Gy2iuPKOSV8Am7RK8vpcjTViODXbL9MbaFWVYpobHmyQmIieXY5byH3JsmLHwe8opForDfp5YxObEVKy8_XkwiVDN7kxE573_p4TaoS45W6D5ZW0-kPgo1FH8EUreBLNlFEXL8EZKGchwIg5CTeV8Fq0WgZThMlqEjJwm4m8rEMyixXwpw2D1memL-So8_QNS8U92?type=png)](https://mermaid.live/edit#pako:eNqFU01v2zAM_SuEjkMSWF7zYQHroegORbHtsNvioGBspjEqS4Yse8uC_PfRX42Tot3JFN_jI59EH0ViUxJKlB493Wf47DCf1mFsYmPsExnvDqDgu4U2jM0ea3p6oUNqfxsGHvuoB7a0s44yU1Rewd35MOA98jDOsVjhqCxbsTY6A1XR9eDvFZvSDoHh1EwMAOtPG5hOb2EYvk2-OmmQKwdrFtladOnXmjkzRwWhhy_gXUUbYB5kJexJp9AbZb0LiXE31uN0giYh_Q71-p64JNmjw8STgxp1ReAooaxuPb2rMNTe__gGVZHy443pY_3LkouK_xScX2Y9hLwGuNWUbi5G66APGo1Ge9ujMfYRXhU8AGoNKe2w0h6Gy2iuPKOSV8Am7RK8vpcjTViODXbL9MbaFWVYpobHmyQmIieXY5byH3JsmLHwe8opForDfp5YxObEVKy8_XkwiVDN7kxE573_p4TaoS45W6D5ZW0-kPgo1FH8EUreBLNlFEXL8EZKGchwIg5CTeV8Fq0WgZThMlqEjJwm4m8rEMyixXwpw2D1memL-So8_QNS8U92)

The nature of key press mechanism brings a fundamental theory of the keyboard interaction - **the pair of "keyup" and "keydown" events**. This pair of events is essential in any keyboard interaction. Hence, the keyboard interaction appears if and only if the pair appears within set of key events.

### Composition Events Mecanics


| <div style="width:15cm">Composition Events Diagram </div> | Explanation |
| ---------------------------------------------------- | ------------- |
|<img src="https://mermaid.ink/img/pako:eNqVVMFu2zAM_RVCx8EJHNlNYgHbYekORbHtUOyyqig0m2mM2ZIhy9myIP8-OVZixXELFL7QfI9P1COhPUlVhoSR2giDt7l40aKcbCmXXEr1jNLoHTD4pqALudyILT7_xl2m_kiL3LvoElj1iANSVVaqzk2upD1KG7bqEw9t4prXVJntySf-OGaumSgzBh7vi8wc6ReulcZcVo1h8Ln_cbAD7ryUvUFTde03lZ9b9cnTBwCPH55gMvkEJ7OOybNzLXJpmA0gr2GDRQadPZZ_QTnXDC0D9pjX7pbyBT7CWhQ1PkEuLcf6Al4B9LpXMucDPHPs3QbiRjdWO90ILVKDGraiaBA0pphvMeNe477MWPONm9pIQx3UV7l5wO33r61LHXxx2OCYbjKvNO-81ligqFsZz-p2nsPprNhbMpXGeqjS7fqYoe_283XV4aKPbsGovZY-sIpdm8L9Zem2v62xi00CUqIuRZ7ZB2LfkjgxGyyRE2bDDNeiKQwnXB4sVTRGPexkSlh71YB0w3NPCmHHNgNSCflTqfJEsr-E7clfwqL5bBouk2RJ6SKe03kYB2RH2CycTZMbSm_iZJ4kNJzRQ0D-HRXC6SJOFlFEaRgl0TKKl4f_PDW1wA?type=png)](https://mermaid.live/edit#pako:eNqVVMFu2zAM_RVCx8EJHNlNYgHbYekORbHtUOyyqig0m2mM2ZIhy9myIP8-OVZixXELFL7QfI9P1COhPUlVhoSR2giDt7l40aKcbCmXXEr1jNLoHTD4pqALudyILT7_xl2m_kiL3LvoElj1iANSVVaqzk2upD1KG7bqEw9t4prXVJntySf-OGaumSgzBh7vi8wc6ReulcZcVo1h8Ln_cbAD7ryUvUFTde03lZ9b9cnTBwCPH55gMvkEJ7OOybNzLXJpmA0gr2GDRQadPZZ_QTnXDC0D9pjX7pbyBT7CWhQ1PkEuLcf6Al4B9LpXMucDPHPs3QbiRjdWO90ILVKDGraiaBA0pphvMeNe477MWPONm9pIQx3UV7l5wO33r61LHXxx2OCYbjKvNO-81ligqFsZz-p2nsPprNhbMpXGeqjS7fqYoe_283XV4aKPbsGovZY-sIpdm8L9Zem2v62xi00CUqIuRZ7ZB2LfkjgxGyyRE2bDDNeiKQwnXB4sVTRGPexkSlh71YB0w3NPCmHHNgNSCflTqfJEsr-E7clfwqL5bBouk2RJ6SKe03kYB2RH2CycTZMbSm_iZJ4kNJzRQ0D-HRXC6SJOFlFEaRgl0TKKl4f_PDW1wA" width="1000" >  | There are two types of each key event. It can be either composed or not composed which is inditifed by the `isComposed` parameter. The composition events occur when browser needs to combine multiple keystrokes or other inputs into a single text composition. For example, if a user is typing a word with an accent or special character, the browser may need to combine several keystrokes to produce the correct character e.g %, ß, Ǣ, ☺, ⓶. Although the structure of the composition events is similar to the simple key press mechanics, the keyboard interaction omits all the composed events. In this case if the event's parameter  `isComposed` is set to `true` the event will **not** be included in the interaction.|


## Diagram of a Keyboard Interaction
[![](https://mermaid.ink/img/pako:eNqVkstO6zAQhl9lNEuUVnFITGIJNlzE0RFs2OFEkWlcatHYVeIUQtV3xw6EBg5nwc4z8_nXP5cdLkwlkWFrhZUXSjw2op5to1znWptSatv0wODWwPuTkyLXK7GV5ZPsy8o8a1e9djG4eAg_uGjkFqbemFZZZTx6fogutw7lx45T2spGLHyyXCqt2pWsGPw5ZGHMAo8Lbw0A-FEBs9kZjC6H5KdlX_luc3TIk2KAv9Y_f0wMM-Cq_bCsH-EUbNPJAiaEG1tjgdOJ4rThqUH4R20p1u1XOaldiyf_s_fTnLyqg7qNb6_0y3TCY6djqoB3hKe_lL4R_YO8Wnft6q_sH4xoqkvXipItz8Y9_PjTS7oFYYC1bGqhKndhO4_naFeyljky96zkUnRrm2Ou9w4VnTV3vV4g82MOsNtUh5tENkwrwI3Q98bUI-RCZDt8QRadzOPjiIZhlqYRySiJAuyRkZDOk4xkEUlSGsZZluwDfB0UwjlNSRTGCc1onBCS0v0bxdoOKA?type=png)](https://mermaid.live/edit#pako:eNqVkstO6zAQhl9lNEuUVnFITGIJNlzE0RFs2OFEkWlcatHYVeIUQtV3xw6EBg5nwc4z8_nXP5cdLkwlkWFrhZUXSjw2op5to1znWptSatv0wODWwPuTkyLXK7GV5ZPsy8o8a1e9djG4eAg_uGjkFqbemFZZZTx6fogutw7lx45T2spGLHyyXCqt2pWsGPw5ZGHMAo8Lbw0A-FEBs9kZjC6H5KdlX_luc3TIk2KAv9Y_f0wMM-Cq_bCsH-EUbNPJAiaEG1tjgdOJ4rThqUH4R20p1u1XOaldiyf_s_fTnLyqg7qNb6_0y3TCY6djqoB3hKe_lL4R_YO8Wnft6q_sH4xoqkvXipItz8Y9_PjTS7oFYYC1bGqhKndhO4_naFeyljky96zkUnRrm2Ou9w4VnTV3vV4g82MOsNtUh5tENkwrwI3Q98bUI-RCZDt8QRadzOPjiIZhlqYRySiJAuyRkZDOk4xkEUlSGsZZluwDfB0UwjlNSRTGCc1onBCS0v0bxdoOKA)

### Note:
* Each [key_code](https://w3c.github.io/uievents/#keys-codevalues) has its own state machine. That means, when there are multiple keyboard interactions happening at the same time. You should expect multiple state machines running at the same time, and each `key_code` allows to a identify the physical key associated with the keyboard event.
***


## States

### `[1]` No entry
The initial state. Either no entry of the key_code has been seen or previous ones have been cancelled.
`key_code_entry_map_` does not contain any entry with key equal to key_code.

### `[2]` Have keydown entry
An intermediate state. In this state, we have seen the `keydown` entry for the current interaction, and are waiting for the matching `keyup` entry.
`key_code_entry_map_` currently contains the `keydown` entry of the interaction that this state machine represent. In this state, `keydown` entry waiting for a **matching** `keyup` entry to finish current interaction, which may or may not show up.

### `[3]` Composition Event
The state indicates that keydown has initiated the composition. Since the composition events are not part of the keyboard interactions this intermediate state holds until the composition is over.

### `[4]` Interaction finished
This is the end of an interaction lifecycle. The `keydown` entry was paired with the corresponding `keyup` entry and the key_code from the `key_code_entry_map_` was errased.


- - - -

## Transitions

### `[5]` keydown

Save the `keydown` key_code value to the key_code_entry_map_.

### `[6]` [isComposing = true] compositionstart
The keydown event initiates the composition session. The `isComposition` parameter is set to true.

### `[7]` [isComposing = true] compositionend
The keydown ends the composition session [Go to Composition Events Mecanics]. The `isComposition` parameter is set to false.

### `[8]` [keyup key_code = keydown key_code] keyup

The transition occurs if the keyup event is fired and there is a matching key_code of a keydown event. In this transition the following steps are executed:
1.Set current key_code be event’s key_code attribute value. 
2.Generate a new interaction id for the keydown-keyup pair (`keydown` and `keyup`). 
3.Delete the key_code of the pair from the `key_code_entry_map_`.

### `[9]` MaybeFlushKeyboardEntries
https://bugs.chromium.org/p/chromium/issues/detail?id=1420716


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
stateDiagram-v2

no_entry : No entry [1]
have_key_down : Have keydown entry [2]
have_key_up : Waiting keyup [3]
interaction_finished: Interaction finished [4]

   [*] --> no_entry
   no_entry --> have_key_down : keydown [5]
   have_key_down --> no_entry : keycancel [6]
   have_key_down --> have_key_up : keyup [7]
   %no_entry --> have_key_up : keyup [7]
   have_key_up --> no_entry : keycancel [6]
   %have_key_down --> interaction_finished : keyup key_code = keydown key_code[8]
   have_key_up --> interaction_finished : keyup key_code = keydown key_code[8] / MaybeFlushKeyboardEntries (cl 403) [9?]
   no_entry --> interaction_finished : keyup key_code = keydown key_code[8]
   interaction_finished --> [*]

```
