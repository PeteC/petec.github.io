---
layout: post
title: "UICollectionView sticky headers"
date: 2015-01-10 21:00
tags: [iOS, development, UICollectionView, Swift]
---

This post will explain how to mimic the 'sticky' behaviour of `UITableView` headers using a `UICollectionView`. To do this we'll implement a subclass of `UICollectionViewFlowLayout` that we will then use to tweak the position of header views as the content scrolls.


## Header view frames

The first problem we'll tackle is calculating where a section's header view should be placed to make it appear 'sticky'.

The header views are always going to be contained somewhere within the section's bounds, so let's start by creating a method that, given a section index, will return the bounding frame for it's content.

This has been written quite verbosely to make it easier to follow:

``` Swift
    private func frameForSection(section: Int) -> CGRect? {

        // Sanity check
        let numberOfItems = collectionView!.numberOfItemsInSection(section)
        if numberOfItems == 0 {
            return nil
        }
        
        // Get the index paths for the first and last cell in the section
        let firstIndexPath = NSIndexPath(forRow: 0, inSection: section)
        let lastIndexPath = numberOfItems == 0 ? firstIndexPath : NSIndexPath(forRow: numberOfItems - 1, inSection: section)
        
        // Work out the top of the first cell and bottom of the last cell
        var firstCellTop = layoutAttributesForItemAtIndexPath(firstIndexPath).frame.origin.y
        let lastCellBottom = CGRectGetMaxY(layoutAttributesForItemAtIndexPath(lastIndexPath).frame)
        
        // Build the frame for the section
        var frame = CGRectZero

        frame.size.width = collectionView!.bounds.size.width
        frame.origin.y = firstCellTop
        frame.size.height = lastCellBottom - firstCellTop

        // Increase the frame to allow space for the header
        frame.origin.y -= headerReferenceSize.height
        frame.size.height += headerReferenceSize.height
        
        // Increase the frame to allow space for an section insets
        frame.origin.y -= sectionInset.top
        frame.size.height += sectionInset.top
        
        frame.size.height += sectionInset.bottom
        
        return frame
    }
```

Now we can find out a section's frame, let's use this information to calculate where withing that frame the section header should be placed. `UICollectionViewLayout` has a method called `layoutAttributesForSupplementaryViewOfKind` that returns the layout attributes for supplementary views. We'll override this method to return a modified frame for our header views.

![Calculating header frame](/images/posts/sticky-headers-frame.png)

``` Swift
override func layoutAttributesForSupplementaryViewOfKind(elementKind: String, atIndexPath indexPath: NSIndexPath) -> UICollectionViewLayoutAttributes! {
    // Get the layout attributes for a standard flow layout
    let attributes = super.layoutAttributesForSupplementaryViewOfKind(elementKind, atIndexPath: indexPath)
    
    // If this is a header, we should tweak it's attributes
    if elementKind == UICollectionElementKindSectionHeader {
        if let fullSectionFrame = frameForSection(indexPath.section) {
            let minimumY = max(collectionView!.contentOffset.y + collectionView!.contentInset.top, fullSectionFrame.origin.y)
            let maximumY = CGRectGetMaxY(fullSectionFrame) - headerReferenceSize.height - collectionView!.contentInset.bottom
            
            attributes.frame = CGRect(x: 0, y: min(minimumY, maximumY), width: collectionView!.bounds.size.width, height: headerReferenceSize.height)
            attributes.zIndex = 1
        }
    }
    
    return attributes
}
```




## Using the updated attributes

`UICollectionViewLayout` has another method that returns layout attributes for header views, `layoutAttributesForElementsInRect`, and we should also override this to return our modified header view frames.

The default implementation of `layoutAttributesForElementsInRect` for `UICollectionViewFlowLayout` will only return attributes for headers where the top of a section is visible. We also need to make sure we include attributes for header views where any part of a section is visible.

``` Swift
override func layoutAttributesForElementsInRect(rect: CGRect) -> [AnyObject]? {
    // Get the layout attributes for a standard UICollectionViewFlowLayout
    var elementsLayoutAttributes = super.layoutAttributesForElementsInRect(rect) as? [UICollectionViewLayoutAttributes]
    if elementsLayoutAttributes == nil {
        return nil
    }
    
    
    // Define a struct we can use to store optional layout attributes in a dictionary
    struct HeaderAttributes {
        var layoutAttributes: UICollectionViewLayoutAttributes?
    }
    var visibleSectionHeaderLayoutAttributes = [Int : HeaderAttributes]()
    
    
    // Loop through the layout attributes we have
    for (index, layoutAttributes) in enumerate(elementsLayoutAttributes!) {
        let section = layoutAttributes.indexPath.section
        
        switch layoutAttributes.representedElementCategory {
        case .SupplementaryView:
            // If this is a set of layout attributes for a section header, replace them with modified attributes
            if layoutAttributes.representedElementKind == UICollectionElementKindSectionHeader {
                let newLayoutAttributes = layoutAttributesForSupplementaryViewOfKind(UICollectionElementKindSectionHeader, atIndexPath: layoutAttributes.indexPath)
                elementsLayoutAttributes![index] = newLayoutAttributes
                
                // Store the layout attributes in the dictionary so we know they've been dealt with
                visibleSectionHeaderLayoutAttributes[section] = HeaderAttributes(layoutAttributes: newLayoutAttributes)
            }
            
        case .Cell:
            // Check if this is a cell for a section we've not dealt with yet
            if visibleSectionHeaderLayoutAttributes[section] == nil {
                // Stored a struct for this cell's section so we can can fill it out later if needed
                visibleSectionHeaderLayoutAttributes[section] = HeaderAttributes(layoutAttributes: nil)
            }
        
        case .DecorationView:
            break
        }
    }
    
    // Loop through the sections we've found
    for (section, headerAttributes) in visibleSectionHeaderLayoutAttributes {
        // If the header for this section hasn't been set up, do it now
        if headerAttributes.layoutAttributes == nil {
            let newAttributes = layoutAttributesForSupplementaryViewOfKind(UICollectionElementKindSectionHeader, atIndexPath: NSIndexPath(forItem: 0, inSection: section))
            elementsLayoutAttributes!.append(newAttributes)
        }
    }
    
    return elementsLayoutAttributes
}
```

There's one last step we need to take. By default `layoutAttributesForElementsInRect` is only called when the new elements are about to be shown in the collection view. We need it to be called whenever the collection view is scrolled. This is a simple change to make, we just need to override `shouldInvalidateLayoutForBoundsChange` to return `true`.

``` Swift
override func shouldInvalidateLayoutForBoundsChange(newBounds: CGRect) -> Bool {
    // Return true so we're asked for layout attributes as the content is scrolled
    return true
}
```

The completed layout class can be viewed [here](https://github.com/petec-blog/CollectionViewStickyHeaders/blob/master/CollectionViewStickyHeaders/StickyHeaderFlowLayout.swift) as part of a full sample project available on [Github](https://github.com/petec-blog/CollectionViewStickyHeaders).
