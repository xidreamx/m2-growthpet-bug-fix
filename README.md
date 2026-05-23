# Metin2 Dupe Fix — Growth Pet & Acce Combine

Closes two dupe bugs. Each fix below shows the original code (BEFORE) and the patched code (AFTER). Copy the AFTER version over the BEFORE.

---

## Fix 1 — `srcs/server/game/growth_pet.cpp`

### 1A. `Feed()` — life-time loop

**BEFORE:**
```cpp
LPITEM item = m_pOwner->GetInventoryItem(dwPos);
if (!item)
{
    packet.subheader = SUBHEADER_PET_FEED_FAILED;
    break;
}

bool bIsFoodItem = true;
```

**AFTER:**
```cpp
LPITEM item = m_pOwner->GetInventoryItem(dwPos);
if (!item)
{
    packet.subheader = SUBHEADER_PET_FEED_FAILED;
    break;
}

if (item->isLocked() || item->IsEquipped() || item->IsExchanging())
{
    packet.subheader = SUBHEADER_PET_FEED_FAILED;
    break;
}

bool bIsFoodItem = true;
```

### 1B. `Feed()` — exp loop

**BEFORE:**
```cpp
LPITEM item = m_pOwner->GetInventoryItem(feedPacket->pos[i]);
if (!item)
{
    packet.subheader = SUBHEADER_PET_FEED_FAILED;
    break;
}

const auto itemType = item->GetType();
```

**AFTER:**
```cpp
LPITEM item = m_pOwner->GetInventoryItem(feedPacket->pos[i]);
if (!item)
{
    packet.subheader = SUBHEADER_PET_FEED_FAILED;
    break;
}

if (item->isLocked() || item->IsEquipped() || item->IsExchanging())
{
    continue;
}

const auto itemType = item->GetType();
```

### 1C. `PremiumFeed()` — validation pass

**BEFORE:**
```cpp
LPITEM pItem = m_pOwner->GetInventoryItem(revivePacket->pos[i]);
if (pItem && pItem->GetType() == ITEM_PET && pItem->GetSubType() == PET_PRIMIUM_FEEDSTUFF)
    materialCount += revivePacket->count[i];
else
```

**AFTER:**
```cpp
LPITEM pItem = m_pOwner->GetInventoryItem(revivePacket->pos[i]);
if (pItem && pItem->GetType() == ITEM_PET && pItem->GetSubType() == PET_PRIMIUM_FEEDSTUFF
    && !pItem->isLocked() && !pItem->IsEquipped() && !pItem->IsExchanging())
    materialCount += revivePacket->count[i];
else
```

### 1D. `PremiumFeed()` — consume loop

**BEFORE:**
```cpp
for (int i = 0; i < PET_REVIVE_MATERIAL_SLOT_MAX; ++i)
{
    LPITEM item = m_pOwner->GetInventoryItem(revivePacket->pos[i]);
    uint32_t dwItemCount = item->GetCount();
```

**AFTER:**
```cpp
for (int i = 0; i < PET_REVIVE_MATERIAL_SLOT_MAX; ++i)
{
    if (requiredMaterialCount == 0)
        break;
    if (revivePacket->count[i] == 0)
        break;

    LPITEM item = m_pOwner->GetInventoryItem(revivePacket->pos[i]);
    if (!item || item->isLocked() || item->IsEquipped() || item->IsExchanging())
        continue;
    if (item->GetType() != ITEM_PET || item->GetSubType() != PET_PRIMIUM_FEEDSTUFF)
        continue;

    uint32_t dwItemCount = item->GetCount();
```

---

## Fix 2 — `srcs/server/game/input_main.cpp`

Apply the same pattern to **every** pet handler that reads an inventory item. Example shown for `PetDetermine` — repeat for `PetAttrChange` (×2 items), `PetRevive`, `PetLearnSkill`, `PetDeleteSkill`, `PetDeleteAllSkill`, `PetNameChange` (×2 items).

**BEFORE:**
```cpp
if (pDetermineItem->GetType() != ITEM_PET || pDetermineItem->GetSubType() != PET_ATTR_DETERMINE)
    return;

if (!ch->GetActiveGrowthPet())
```

**AFTER:**
```cpp
if (pDetermineItem->GetType() != ITEM_PET || pDetermineItem->GetSubType() != PET_ATTR_DETERMINE)
    return;
if (pDetermineItem->isLocked() || pDetermineItem->IsEquipped() || pDetermineItem->IsExchanging())
    return;

if (!ch->GetActiveGrowthPet())
```

---

## Fix 3 — `srcs/server/game/char.cpp` — `RefineAcceMaterials()`

### 3A. Same-item / null guard

**BEFORE:**
```cpp
LPITEM* pkItemMaterial = GetAcceMaterials();
if (!pkItemMaterial)
{
    return;
}

uint32_t dwItemVnum, dwMinAbs, dwMaxAbs;
```

**AFTER:**
```cpp
LPITEM* pkItemMaterial = GetAcceMaterials();
if (!pkItemMaterial)
{
    return;
}

if (!pkItemMaterial[0] || !pkItemMaterial[1] || pkItemMaterial[0] == pkItemMaterial[1])
{
    return;
}

uint32_t dwItemVnum, dwMinAbs, dwMaxAbs;
```

### 3B. Success path

**BEFORE:**
```cpp
uint16_t wCell = pkItemMaterial[0]->GetCell();

ITEM_MANAGER::instance().RemoveItem(pkItemMaterial[0], "COMBINE (REFINE SUCCESS)");
ITEM_MANAGER::instance().RemoveItem(pkItemMaterial[1], "COMBINE (REFINE SUCCESS)");

pkItem->AddToCharacter(this, TItemPos(INVENTORY, wCell));
```

**AFTER:**
```cpp
uint16_t wCell = pkItemMaterial[0]->GetCell();

LPITEM pkAcceMat0 = pkItemMaterial[0];
LPITEM pkAcceMat1 = pkItemMaterial[1];
pkItemMaterial[0] = nullptr;
pkItemMaterial[1] = nullptr;
if (pkAcceMat0)
    ITEM_MANAGER::instance().RemoveItem(pkAcceMat0, "COMBINE (REFINE SUCCESS)");
if (pkAcceMat1)
    ITEM_MANAGER::instance().RemoveItem(pkAcceMat1, "COMBINE (REFINE SUCCESS)");

pkItem->AddToCharacter(this, TItemPos(INVENTORY, wCell));
```

### 3C. Fail path

**BEFORE:**
```cpp
PointChange(POINT_GOLD, -llPrice);
DBManager::instance().SendMoneyLog(MONEY_LOG_REFINE, pkItemMaterial[0]->GetVnum(), -llPrice);

ITEM_MANAGER::instance().RemoveItem(pkItemMaterial[1], "COMBINE (REFINE FAIL)");

if (lVal == 4)
    ChatPacket(CHAT_TYPE_INFO, LC_TEXT("New absorption rate: %d"), pkItemMaterial[0]->GetSocket(ACCE_ABSORPTION_SOCKET));
else
    ChatPacket(CHAT_TYPE_INFO, LC_TEXT("Failed!"));

LogManager::instance().AcceLog(GetPlayerID(), GetX(), GetY(), dwItemVnum, 0, 0, 0, 0);

pkItemMaterial[1] = nullptr;
```

**AFTER:**
```cpp
PointChange(POINT_GOLD, -llPrice);
DBManager::instance().SendMoneyLog(MONEY_LOG_REFINE, pkItemMaterial[0]->GetVnum(), -llPrice);

LPITEM pkAcceFailMat1 = pkItemMaterial[1];
pkItemMaterial[1] = nullptr;
if (pkAcceFailMat1)
    ITEM_MANAGER::instance().RemoveItem(pkAcceFailMat1, "COMBINE (REFINE FAIL)");

if (lVal == 4)
    ChatPacket(CHAT_TYPE_INFO, LC_TEXT("New absorption rate: %d"), pkItemMaterial[0]->GetSocket(ACCE_ABSORPTION_SOCKET));
else
    ChatPacket(CHAT_TYPE_INFO, LC_TEXT("Failed!"));

LogManager::instance().AcceLog(GetPlayerID(), GetX(), GetY(), dwItemVnum, 0, 0, 0, 0);
```

### 3D. Absorb path

**BEFORE:**
```cpp
ITEM_MANAGER::instance().RemoveItem(pkItemMaterial[1], "ABSORBED (REFINE SUCCESS)");
pkItemMaterial[1] = nullptr;

ITEM_MANAGER::instance().FlushDelayedSave(pkItemMaterial[0]);
```

**AFTER:**
```cpp
LPITEM pkAcceAbsorbed = pkItemMaterial[1];
pkItemMaterial[1] = nullptr;
if (pkAcceAbsorbed)
    ITEM_MANAGER::instance().RemoveItem(pkAcceAbsorbed, "ABSORBED (REFINE SUCCESS)");

ITEM_MANAGER::instance().FlushDelayedSave(pkItemMaterial[0]);
```

---

