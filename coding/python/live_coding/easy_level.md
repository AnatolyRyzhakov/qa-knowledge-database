1. Find the smallest integer in a given array:
   - Given ```[34, 15, 88, 2]``` function will return ```2```
   - Given ```[34, -345, -1, 100]``` function will return ```-345```
```py
# First solution
def find_smallest_int(array):
    smallest = array[0]
    for integer in array:
        if integer < smallest:
            smallest = integer
    return smallest

# Second solution
def find_smallest_int(array):
    return min(array)

# Third solution
def find_smallest_int(array):
    arr.sort()
    return array[0]
```

2. Calculate the average of the numbers in a given array (with zero division protection)
```py
# First solution
def find_average(numbers):
   if not numbers:
      return 0

   total = 0
   for num in numbers:
      total += num

   return total / len(numbers)

# Third solution
def find_average(numbers):
   return sum(numbers) / len(numbers) if numbers else 0

# Second solution
def find_average(numbers):
   try:
      return sum(numbers) / len(numbers)
   except ZeroDivisionError:
      return 0
```
