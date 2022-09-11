---
title: "Svelte: Warn user when caps lock is active"
date: "2022-05-17"
tags: [svelte,html5,webdev,typescript]
lang: "en"
---

Useful when webpage contains password inputs

```svelte
<script lang="ts">
    let warnCapsLockOn: boolean = false

    function checkCapsLock(e: KeyboardEvent) {
        warnCapsLockOn = !!e.getModifierState("CapsLock")
    }
</script>

<svelte:body on:keyup={checkCapsLock}/>
    
{#if warnCapsLockOn}
    <div class="text-danger">Attention, caps lock on!</div>
{/if}
```
