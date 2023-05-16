## Complexity
Time: O(m+n), Space: m+n

## Solution
Create a new array of all the elements, maintaining a sorted order using two pointers. 
Return either the middle element, if the number of elements is uneven.If not, return the average of the two middle elements. 

### Create sorted array

Create an array that holds the total elements of nums1 and nums2, i.e. of size m+n
Initialize a pointer for each array to 0
Copy all the elements in sorted order to the new array by taking the smallest element that is currently pointed and incrementing that pointer, repeating untill all elements have been copied

### Compute result
If m+n is even:
	return avg of the two middle elements 
Else:
	return middle element