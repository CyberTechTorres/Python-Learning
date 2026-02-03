# [Andres's Conceptuliazing Python Home Lab]() 

<img width="700" height="415" alt="image" src="https://github.com/user-attachments/assets/d6875007-a771-49e0-984e-5deeee386897" />

## Platforms and Languages Leveraged
- Windows 11
- VSCode
- Python
- HarvardX CS50P website

##  Scenario
Python is crucial in cybersecurity because it excels at automating repetitive tasks like log analysis, vulnerability scanning, and threat hunting—freeing analysts for high-value work.

Here I will begin by going through the provided problomatic sceanarios HarvardX CS50P provides in Problem Set 2

---


### 1. `Vanity Plates` Problem Set

Implement a program that prompts the user for a vanity plate and then output "Valid" if meets all of the requirements or "Invalid" if it does not. Assume that any letters in the user’s input will be uppercase. Structure your program per the below, wherein is_valid returns True if s meets all requirements and False if it does not. Assume that s will be a str. You’re welcome to implement additional functions for is_valid to call (e.g., one function per requirement).<br/>

**Requirements:**<br/>
-All vanity plates must start with at least two letters.<br/>

-Vanity plates may contain a maximum of 6 characters (letters or numbers) and a minimum of 2 characters.<br/>

-Numbers cannot be used in the middle of a plate; they must come at the end. For example, AAA222 would be an acceptable … vanity plate; AAA22A would not be acceptable. The first number used cannot be a ‘0’<br/>

-No periods, spaces, or punctuation marks are allowed.<br/>

**Script baseline used to resolve**

```kql
def main():
    plate = input("Plate: ")
    if is_valid(plate):
        print("Valid")
    else:
        print("Invalid")


def is_valid(s):
    ...


main()
```

My key focus here is to provide scripting details for what function "is_valid()" does.<br/>
We want to keep def main() untoched therefore it provides what we need already which is printing a result when it receives a Boolean return value of "True" or "False".<br/>

def main() will call out to is_valid() by its "if is_valid(plate):" line. Which checks to see if input variable Plate is True or False. <br/>

def is_valid(s) determines True or False<br/>

Lets verify first if the length of the variable is 2 characters min and max 6 characters<br/>
It begins by breaking down the length of the input variable "plate" using len() <br/>

```
if len(s) < 2 or len(s) > 6:
        return False
```
If length of "s" which is plates variable is Less than 2 OR length of "s" is Greater Than 6 we will return "False" <br/>
If "False" is received the entire "is_valid(s)" function outputs False. <br/>

If its True we move on to next script requirment. <br/>
Lets check if the first 2 letters is alphabet.<br/>
We need to specfify where in "s" are we look at to be considered. <br/>
Every variable has a postion from 0 - (the last postion contained in the syntax) <br/>
We want to look at only the first 2 postions. <br/>
```s[:2]``` Looks from the position of 0 - 1. Position 2 is stopped and not considered.<br/>
We will add ```.isalpha()```  which is pythons built in function to check if its only alphabet.<br/>
***Result:***
```
if not s[:2].isalpha():
        return False
```

Next lets add in the requirement that fulfills No periods, spaces, or punctuation marks are allowed.<br/>
Luckily theres a python built in function where it checks if a variable is only alphanumeric.<br/>
```.isalunm()```
Add a "if not" in front of the variable and it checks if its not alphanumeric. <br/>
```
 if not s.isalnum():
        return False
```
Next is the hardest part. <br/>
We will check to make sure variable doesnt begin with a number after its sequence of strings then check to see the variable continues to end in numbers once it begins its number sequence.<br/>
This fulfills the requirment listed as: Numbers cannot be used in the middle of a plate; they must come at the end. For example, AAA222 would be an acceptable … vanity plate; AAA22A would not be acceptable. The first number used cannot be a ‘0’.<br/>

We need to break down the variable into positions. Each position representing the single string in respect to its position.<br/>
These positions we will name it as "i"<br/>
Using the ```len()``` to represent each sytnax in the string <br/>
Using the ```range()``` built in function, puts each len() into positions <br/>
```
for i in range(len(s)):
```
So now we can create a condition under this...<br/>
When we detect when i is a digit we will stop and check for True or False statements<br/>
When this is looping through the entire variable, We need to verify when the first digit is identified and make sure its NOT 0.<br/>
```
for i in range(len(s)):   
        if s[i].isdigit(): 
```
As its looping through each position which represents "i" within the provided word in variable "s" we check if i within s is a digit. <br/>
If it is... now we implement ```if s[i] == "0"```<br/>
We will return False if it is in fact true that `s[i] == 0.`<br/>

```
for i in range(len(s)):   
        if s[i].isdigit():    
            if s[i] == "0":   
                return False

```
Now if "s" is in fact a digit but it doesnt represent a 0 in its first position we will next look to make sure it ends in digits.<br/>
Within the ```if s[i].isdigit():``` loop we will add ```return all(c.isdigit() for c in s[i:])```<br/>
For whichever position "i" is currently at and for the remaindor of the postions which represnets `s[i:]`<br/>
We will label those positions as "c" as indicated by `for c in s[i:]`<br/>
Now we check if all those c positions are in fact digits by adding in the `c.isdigit()`<br/>
The all() python built in function considers everything within the ( ) <br/>
return `all(c.isdigit() for c in s[i:]` returns if the c.isdigit() is True or False. <br/>
Everything within c.isdigit() must be true to return true. 1 single false in any position returns false.<br/>
Result: 
```
 for i in range(len(s)):   # range() turns the s into integers that represent i
        if s[i].isdigit():    # You have to index into s to make it check for a string
            if s[i] == "0":   
                return False
            return all(c.isdigit() for c in s[i:])
```
But everything together and we have our solution where it check if statments to verify if variable returns "True"

```
def main():
    plate = input("Plate: ")
    if is_valid(plate):
        print("Valid")
    else:
        print("Invalid")


def is_valid(s):
    if len(s) < 2 or len(s) > 6:
        return False
    
    if not s[:2].isalpha():
        return False
    
    if not s.isalnum():
        return False
    
    for i in range(len(s)):   
        if s[i].isdigit():    
            if s[i] == "0":   
                return False
            return all(c.isdigit() for c in s[i:])
    return True
    

main()

```

















