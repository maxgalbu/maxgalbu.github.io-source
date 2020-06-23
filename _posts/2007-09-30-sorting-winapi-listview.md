---
title: WinAPI Listview tutorials - How to sort a listview
layout: post
categories: [WinAPI, Programming]
archived: true
---

<span class="alert y">These are very old tutorials I made when I was testing Windows API back in 2007 when I was young and full of energy. Please be aware that they can be <ins>waaaaay</ins> out of date!</span>

## How to sort a listview

### Download

*   [Demo (executable)](/assets/download/columnsortimage/columnsortimage_demo.rar)
*   [Source (plus a C::B project)](/assets/download/columnsortimage/columnsortimage_example.rar)
*   [Up Arrow BMP](/assets/download/columnsortimage/uparrow.bmp)
*   [Down Arrow BMP](/assets/download/columnsortimage/downarrow.bmp)

Sort a listview is easy. Although you can manually move items, change the items' image and things like that, Windows helps you by providing two messages to be sent to the listview: [`LVM_SORTITEMS`](http://msdn2.microsoft.com/en-us/library/bb761227.aspx){:target="_blank"} and [`LVM_SORTITEMSEX`](http://msdn2.microsoft.com/en-us/library/bb761228.aspx){:target="_blank"} (which you can easily use through the macros [`ListView_SortItems`](http://msdn2.microsoft.com/en-us/library/ms671382.aspx){:target="_blank"} and [`ListView_SortItemsEx`](http://msdn2.microsoft.com/en-us/library/bb761228.aspx){:target="_blank"}).  

Both messages shares a callback function, whose prototype is:

```c
int CALLBACK CompareFunc(LPARAM lParam1, LPARAM lParam2, LPARAM lParamSort);
```

This callback function is where you decide if an item (whose date is in the **lParam1** parameter) must go first or after another item (**lParam2** parameter). The third parameter of the callback function is the parameter you pass in your call to `SendMessage(LVM_SORTITEMS(EX))` or `ListView_SortItems(Ex)`. You decide the value of this parameter. The procedure goes like this:

1.  You send `LVM_SORTITEMS` or `LVM_SORTITEMSEX` to the listview, passing the callback function you created and a parameter you want
2.  Windows calls for each couple of items your callback functions
3.  Inside the callback function, you do the comparison and return
    *   -1: if the first item should precede the second
    *   0: if the two items are the same
    *   1: if the second item should precede the first

### The difference between `LVM_SORTITEMS` and `LVM_SORTITEMSEX`

The two messages share the same callback and both requires an user-defined variable. The difference is that the first (`LVM_SORTITEMS`) pass to the callback functions the user-defined value you used in the **lparam** member of the **LVITEM** structure. This is useful only in rare cases, that is why you'll usually want to use the `LVM_SORTITEMSEX`. This message makes Windows send the index of the item to the callback, so you can get the text of the item using `LVM_GETITEMTEXT` message or any other data through the `LVM_GETITEM` message.  

### The callback function

As said before, the older versions of Windows needs an image to be set manually in the header: two images, actually, a **down** and an **up arrow** bitmaps. These two bitmaps are available in the download section of this page. It's your choice where to put them: i've included them in the executable and i use LoadImage and IDs to load those images. Change it if you put them elsewhere.  

Here's the complete function code:

```c
void ListView_SetHeaderSortImage(HWND listView, int columnIndex, BOOL isAscending)
{
    HWND header = ListView_GetHeader(listView);
    BOOL isCommonControlVersion6 = IsCommCtrlVersion6();
    int columnCount = Header_GetItemCount(header);
    
    for (int i = 0; i<columnCount; i++)
    {
        HDITEM hi = {0};
        hi.mask = HDI_FORMAT | (isCommonControlVersion6 ? 0 : HDI_BITMAP);
        Header_GetItem(header, i, &hi);
        
        //Set sort image to this column
        if (i == columnIndex)
        {
            if (isCommonControlVersion6)
            {
                hi.fmt &= ~(HDF_SORTDOWN|HDF_SORTUP);
                hi.fmt |= isAscending ? HDF_SORTUP : HDF_SORTDOWN;
            }
            else
            {
                UINT bitmapID = isAscending ? IDB_UPARROW : IDB_DOWNARROW;
                
                //If there's a bitmap, let's delete it.
                if (hi.hbm)
                    DeleteObject(hi.hbm);
                
                hi.fmt |= HDF_BITMAP|HDF_BITMAP_ON_RIGHT;
                hi.hbm = (HBITMAP)LoadImage(GetModuleHandle(NULL), MAKEINTRESOURCE(bitmapID), IMAGE_BITMAP, 0,0, LR_LOADMAP3DCOLORS);
            }
        }
        //Remove sort image (if exists)
        //from other columns.
        else
        {
            if (isCommonControlVersion6)
                hi.fmt &= ~(HDF_SORTDOWN|HDF_SORTUP);
            else<
            {
                //If there's a bitmap, let's delete it.
                if (hi.hbm)
                    DeleteObject(hi.hbm);
                
                hi.mask &= ~HDI_BITMAP;
                hi.fmt &= ~(HDF_BITMAP|HDF_BITMAP_ON_RIGHT);
            }
        }
        
        Header_SetItem(header, i, &hi);
    }
}
```

### The function, explained

The function has this prototype:

```c
void ListView_SetHeaderSortImage(HWND listView, int columnIndex, BOOL isAscending);
```

So, whenever you call this function, you must pass:

*   the listview window handle
*   the index of the column that is sorted
*   whether or not the order is ascending (up arrow displayed) or descending (down arrow displayed)

The `LVN_COLUMNCLICK` notification handler should be the right place to put this function.  

Inside the function now:

```c
HWND header = ListView_GetHeader(listView);  
BOOL isCommonControlVersion6 = IsCommCtrlVersion6();
```

The `LVCOLUMN` structure is useless, because it doesn't let you set all the header's features. So i'm going to use the header messages (`HDM_X`) sent to the header window, obtained through `ListView_GetHeader`. I store the BOOL that is TRUE or FALSE whether the application has commctrl version 6 loaded, to avoid unnecessary overhead by calling the function more than once inside this function.  

```c
int columnCount = Header_GetItemCount(header);
for (int i = 0; i<columnCount; i++)
{
    //...
}
```

The function doesn't store neither require from you the last ordered column index, so i need to iterate through all the existing columns to add the image to the correct column index and remove it from all the other column. And, since there are always few columns, this loop doesn't halt your program.  

```c
HDITEM hi = {0};
hi.mask = HDI_FORMAT | (isCommonControlVersion6 ? 0 : HDI_BITMAP);
Header_GetItem(header, i, &hi);
```

Here i retrieve the column props from the header through `Header_GetItem`. With commctrl 6, the only thing that changes is the format; without it i need to retrieve and set the bitmap too. So,

*   version >= 6: `HDI_FORMAT`
*   version < 6: `HDI_FORMAT|HDI_BITMAP`

```c
if (i == columnIndex)
{
    if (isCommonControlVersion6)
    {
        hi.fmt &= ~(HDF_SORTDOWN|HDF_SORTUP);
        hi.fmt |= isAscending ? HDF_SORTUP : HDF_SORTDOWN;
    }
    else
    {
        UINT bitmapID = isAscending ? IDB_UPARROW : IDB_DOWNARROW;
        
        //If there's a bitmap, let's delete it.
        if (hi.hbm)
            DeleteObject(hi.hbm);
        
        hi.fmt |= HDF_BITMAP|HDF_BITMAP_ON_RIGHT;
        hi.hbm = (HBITMAP)LoadImage(GetModuleHandle(NULL), MAKEINTRESOURCE(bitmapID), IMAGE_BITMAP, 0,0, LR_LOADMAP3DCOLORS);
    }
}
```

The column index corresponds to the index you passed to the function, so this column is being sorted. As before, two ways:

*   version >= 6
    *   since i don't know which flag was set before, i remove the both of them
    *   set the correct format flag for the order basing upon the value of **isAscending**: `HDF_SORTUP` or `HDF_SORTDOWN`. Windows will draw the order image in the header.
*   version < 6
    *   delete the old bitmap if it exists
    *   append the format flags: `HDF_BITMAP` (again :) and `HDF_BITMAP_ON_RIGHT` (since the bitmap should be to the right of the text)
    *   load and set the correct bitmap basing on the value of `isAscending` with LoadImage and the special flag `LR_LOADMAP3DCOLORS`. This flag replace the gray color ( RGB(192,192,192) ) with the corresponding system color (`COLOR_3DFACE`) that is also the background color of each column in the header.

```c
else
{
    if (isCommonControlVersion6)
        hi.fmt &= ~(HDF_SORTDOWN|HDF_SORTUP);
    else
    {
        //If there's a bitmap, let's delete it.
        if (hi.hbm)
            DeleteObject(hi.hbm);
        
        hi.mask &= ~HDI_BITMAP;
        hi.fmt &= ~(HDF_BITMAP|HDF_BITMAP_ON_RIGHT);
    }
}
```

All the other column indexex aren't being sorted, so i remove the image/flags from them (even if they don't have it).  

```c
Header_SetItem(header, i, &hi);
```

Finally, i apply the changes through `Header_SetItem` and the modified `HDITEM structure`.  

