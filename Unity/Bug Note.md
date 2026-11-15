## Bug 001
### Bất cập trong `LoadHotbarItems()` (Lỗi `Destroy()` trong Unity)


```
// ĐOẠN CODE CŨ:
for (int i = 0; i < numOfSlots; i++)
{
    Slot slot = hotbarPanel.transform.GetChild(i).GetComponent<Slot>();
    if (slot != null) Destroy(slot.gameObject);
}

for (int i = 0; i < numOfSlots; i++)
{
    Instantiate(hotbarSlotPrefab, hotbarPanel.transform);
}
```

- **Lỗi:** Hàm `Destroy(slot.gameObject)` không xóa object ngay lập tức mà phải chờ **cuối frame**. Khi bạn gọi ngay vòng lặp `Instantiate` tiếp theo trong cùng 1 frame, các `Slot` mới sẽ bị xếp nối tiếp phía sau các `Slot` cũ (chưa kịp xóa), làm cho `hotbarPanel.transform.GetChild(saveData.slotIndex)` lấy **sai vị trí slot**.
    
- **Sửa lại:** Thay vì xóa đi tạo lại toàn bộ `Slot`, bạn chỉ cần **dọn dẹp các item con đang nằm bên trong từng Slot**:
    
    C#
    
    ```
    public void LoadHotbarItems(List<InventorySaveData> saveDataList)
    {
        // 1. Xóa các Item cũ nằm trong từng Slot (nếu có)
        foreach (Transform slotTransform in hotbarPanel.transform)
        {
            Slot slot = slotTransform.GetComponent<Slot>();
            if (slot != null && slot.currentItem != null)
            {
                Destroy(slot.currentItem);
                slot.currentItem = null;
            }
        }
    
        // 2. Load và gắn lại Item mới vào đúng Slot
        foreach (InventorySaveData saveData in saveDataList)
        {
            if (saveData.slotIndex < hotbarPanel.transform.childCount)
            {
                Slot slot = hotbarPanel.transform.GetChild(saveData.slotIndex).GetComponent<Slot>();
                GameObject itemPrefab = itemDictionary.GetItemPrefabById(saveData.itemID);
                if (itemPrefab != null)
                {
                    GameObject item = Instantiate(itemPrefab, slot.transform);
                    item.GetComponent<RectTransform>().anchoredPosition = Vector2.zero;
                    slot.currentItem = item;
                }
            }
        }
    }
    ```

## Bug 002

```
public void CreateChoiceButtons(DialogueChoice[] choices, Action<int> onChoiceSelected)
{
    ClearChoices();
    for (int i = 0; i < choices.Length; i++)
    {            
        for (int j = 0; j < choices[i].choices.Length; j++)
        {
            GameObject buttonObj = Instantiate(choiceButtonPrefab, choicesContainer);
            TMP_Text buttonText = buttonObj.GetComponentInChildren<TMP_Text>();
            buttonText.SetText(choices[i].choices[j]);
            Button button = buttonObj.GetComponent<Button>();
            int nextDialogueIndex = choices[i].nextDialogueIndices[j];                
            button.onClick.AddListener(() => onChoiceSelected(nextDialogueIndex));
        }            
    }
}
```
### Root cause
The button callback captured the loop variables i and j:

```
button.onClick.AddListener(() => onChoiceSelected(choices[i].nextDialogueIndices[j]));
```

When the button was clicked, both loops had already completed, so the callback accessed invalid indices and raised IndexOutOfRangeException.

### Change
Captured the valid next-dialogue index during button creation:

```
int nextDialogueIndex = choices[i].nextDialogueIndices[j];
button.onClick.AddListener(() => onChoiceSelected(nextDialogueIndex));
```

