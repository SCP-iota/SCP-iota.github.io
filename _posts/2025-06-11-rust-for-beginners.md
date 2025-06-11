---
layout: blog_post
category: software
title: Rust Could be a Good Beginner Language
description: "It's often assumed that Rust's low-level nature and fundamental differences from common languages make it unsuitable for beginners, but I suggest that Rust's apparent difficulty is instead caused by our familiarity with other languages."
---

If I had a nickel for every question along the lines of "What programming language should I learn as a beginner?" posted online, I'd pay for these people's Rust courses. I know it sounds counterintuitive; after all, Rust is the new C++, not the new Python, right? Yet, when I stop and think about which aspects of Rust made it difficult to get used to, it wasn't the usual stumbling blocks of low-level languages. That's not just because I already knew C++, either; rather, it felt like Rust actually *eased* many of those pitfalls, while the parts that tripped me up did so because they felt entirely new and incomparable to the languages I had already learned. The turning point that made it suddenly easier to understand Rust was when I realized that I needed to stop trying to map concepts from other languages into Rust syntax and start learning Rust *as a system* rather than merely *as a language*.

A beginner learning their first programming language, on the other hand, is not already familiar with the ways of thinking that we tend to hold in our minds when we set out to add another language to our collection. These newbies are tasked with learning the concepts and the language simultaneously, and, despite the difficulty that *adds*, that process can also *ease* learning when the language relies on concepts that differ from those the rest of us have already learned.

Rust, for all the talk of its high-and-mighty enlightenment that mere mortals have too much "skill issue" to comprehend, might be an ideal language for absolute beginners to learn programming concepts in a way that better helps them eventually understand computing and move on to other languages.

## "Forget Everything You Know"

Imagine a beginner to programming - let's call him Kevin. Kevin has just finished learning about structures or classes, depending on which language he is learning, and now wants to try passing an instance to a function. If he was learning, say, Python, he might try something like this:

```python
class Person:
    def __init__(self, name, age):
        self.name = name
        self.age = age

def increase_age(person):
    person.age += 1
    print(f'{person.name} has aged')

ada = Person('Ada Lovelace', 35)
increase_age(ada)
print(f'new age: {ada.age}')
```

Kevin has just figured out that objects in Python are passed by reference - when he changes a property of an object that came from a parameter, it changes the same object that was created by the caller. This can be just as much of a pitfall as a feature for a beginner:

```
olympics_2010_ice_dancing_placed_nations = ['United States', 'Russia', 'Canada']
olympics_2010_ice_dancing_nation_points = [215.74, 207.64, 221.57]

def display_nations(nations):
    nations.sort()
    print('Nations that placed:')

    for nation in nations:
        print(f'* {nation}')

display_nations(olympics_2010_ice_dancing_placed_nations)
canada_index = olympics_2010_ice_dancing_placed_nations.index('Canada')
print(f'Canada scored a total of {olympics_2010_ice_dancing_nation_points[canada_index]} points')
```

Yes, I know it's messy, but it's messy in the way an inexperienced programmer's code might be. There's a bigger problem here: according to the output of this program, `Canada scored a total of 215.74 points`, but that is incorrect - Canada scored 221.57 points as shown in the lists.

The issue is that the `.sort()` changes the list of nations *in-place*, which takes it out of alignment with the score list. This could've been avoided using `.sorted()`, but a beginner might not think about how the changes they make to a list from a parameter could affect other code because, the way they might see it, the parameter falls out of scope right after and it doesn't matter.

Now consider an alternate universe where our hypothermic Kevin is instead learning Rust. That same program might look like this:

```rust
fn display_nations(mut nations: Vec<&str>) {
    nations.sort();
    println!("Nations that placed:");
    
    for nation in nations {
        println!("* {}", nation);
    }
}

fn main() {
    let olympics_2010_ice_dancing_placed_nations = vec!["United States", "Russia", "Canada"];
    let olympics_2010_ice_dancing_nation_points = [215.74, 207.64, 221.57];
    
    display_nations(olympics_2010_ice_dancing_placed_nations.clone());
    let canada_index = olympics_2010_ice_dancing_placed_nations.iter().position(|&n| n == "Canada").unwrap();
    println!("Canada scored a total of {} points", olympics_2010_ice_dancing_nation_points[canada_index]);
}
```

Grumbles about using `Vec<&str>` instead of `Vec<impl AsRef<str> + Ord + Display>` aside, I can hear the angry readers already: "That's cheating! You snuck a `.clone()` in there to fix the issue!" The thing is, though, *Rust's borrow checker made me put that there.* Without the `.clone()`, the compiler would complain about the line that determines `canada_index` because the array of nations has already been moved by that point. If the beginner had instead fixed that by making `display_nations` take a reference rather than taking the vector by value, the compiler would've complained that the array cannot be borrowed as mutable because it is not declared as mutable, and with that, the programmer would become aware that there is a risk of changing something that needs to stay in the same order.

The point is that Rust's concepts of reference vs. value passing, ownership, and immutability make the programmer aware of things that would remain in the background in a language like Python, and that's especially bad for beginners who might not think of those things and shoot themselves in the foot.

This raises the question of whether learning about things like ownership and references makes Rust too hard to learn as a first language - and I think the answer is no. Ownership may trip up programmers with experience in other languages who move on to learning Rust, but a beginner learning Rust first would learn it as a basic concept just like any other. In a way, the idea of value ownership is a very tangible and intuitive part of Rust that I think many beginners will have no issue visualizing. It's the ones of us who think we already have the concepts we need that become confused.

## Too Verbose?

There is a reason Python is a popular first language, and it's not just because of the machine learning boom. As a dynamic language, Python is semantically simple with its lack of type declarations. Compared to learning, say, Java as a first language, Python keeps that part out of the visual space so that the programmer can focus on the flow of the program. As we've seen from complaints about runtime errors, though, a dynamic type system can create pitfalls of its own. A nice middle ground is type inference, which has the simplicity benefits of no explicit type declarations at the same time as compile-time type checking. This can be seen with the likes of C# and Kotlin, and modern Java's `var` and C++'s `auto`, but these basic type inference solutions that were shoehorned into existing languages are only useful in some situations and leave quite a bit of code still requiring explicit types. Rust, however, has powerful type inference. Take for example this simple Rust program:

```rust
use std::io::{stdin, stdout, Write};

fn main() {
    let input = stdin();
    let mut output = stdout();
    let mut numbers = vec![];
    
    println!("Enter some numbers, one per line. Leave blank to stop:");
    
    loop {
        print!("-> ");
        output.flush().unwrap();
        let mut line = String::new();
        input.read_line(&mut line).unwrap();
        let value = line.trim();
        
        if value.is_empty() {
            break;
        }
        
        if let Ok(num) = value.parse::<f32>() {
            numbers.push(num);
        } else {
            println!("That's not a valid number.");
        }
    }
    
    println!("Sum of all numbers: {}", numbers.iter().sum::<f32>());
}
```

Notice how there are only two explicit type annotations in all this - the ones for `parse` and `sum`. All of the variables have inferred types. Notably, even the vector has an inferred item type, which Rust detects from the code later on. This larger scope for type inference compared to most languages allows code to *look* more like the simple, dynamically typed Python despite benefitting from compile-time type checks.

## A Solid Foundation

Many of us started learning programming with simpler languages like Python and then worked up to the likes of C++, and many of us remember the difficulties of getting used to lower-level concepts we had never seen before. If a beginner started with Rust and learned the basics of programming while being exposed to concepts like ownership, moving, and references, even if they had no understanding of the underlying memory it would likely be easier for them to learn C++. Learning Rust builds a foundation that may prepare someone to learn a difficult low-level language even if they do not learn those advanced concepts in Rust itself. We as people who learned Rust after other languages are facing the difficulties of a top-down learning order, so let's prepare the next set of programmers to go from the bottom up.