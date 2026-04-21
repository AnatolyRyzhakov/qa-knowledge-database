1. Find smallest integer:
   - Given ```[34, 15, 88, 2]``` function will return ```2```
   - Given ```[34, -345, -1, 100]``` function will return ```-345```
```py
# First solution
def find_smallest_int(arr):
    smallest = arr[0]
    for integer in arr:
        if integer < smallest:
            smallest = integer
    return smallest

# Second solution
def find_smallest_int(arr):
    return min(arr)
```
