---
layout:     post
title:      "Developing APIs in Rust"
date:       2022-02-21 00:00:00
author:     Jon Winsley
comments:   true
summary:    "Rust is a language designed for the next forty years. Let's try building an API!"
---

Rust has been skyrocketing in popularity over the last five years or so. Stack Overflow's Developer Survey voted it the [most loved language](https://insights.stackoverflow.com/survey/2021#technology-most-loved-dreaded-and-wanted) every year since 2016. And it's designed to be around for the long haul: Rust has been described as [a language for the next 40 years](https://www.youtube.com/watch?v=A3AdN7U24iU).

Rust gives you a lot of low-level control while preventing many dangerous security issues, making it an ideal candidate to replace systems languages like C or C++. But it's also a high-level language with a growing ecosystem of tools and crates to accelerate application development. As an example, we'll use [Rocket](https://rocket.rs/) to develop a simple, efficient API server.

## Setting Up the Workspace

Visual Studio Code's [devcontainers](https://code.visualstudio.com/docs/remote/create-dev-container) are a great way to spin up a self-contained dev environment with compilers and extensions pre-configured. You can do the work yourself, or you can just have Code spin up an environment for you:

![Adding devcontainer config files automatically](assets/rust-create-devcontainer.png)

After opening your workspace in the devcontainer, you'll have the Rust `cargo` tool and some useful extensions. (Side note: I had to uninstall and re-install the `rust-analyzer` extension to get it to work; your mileage may vary.)

## Planning Our API

Let's make a simple word game. We'll submit our guess, and the API will return a response that tells us which letters in our guess were correct. We can specify the ID for a word in our request, to guess against the same word again; otherwise, the API will select a random word for us, and return its ID in the response:

```json
// Request
{
    "guess": "whale",
    "word": 0 // optional - will use random word if not provided
}

// Response
{
    "results": ["CORRECT", "CORRECT", "ALMOST", "WRONG", "WRONG"],
    "word": 0,
    "win": false
}
```

## Setting up Rocket

First things first, let's initialize our Rust project with Cargo:

```plaintext
cargo init wurtle
cd wurtle
```

This creates `Cargo.toml`, which contains metadata about our project and its dependencies, and a `main.rs` file for us to start with. You can run this and see that it compiles and executes:

```plaintext
$ cargo run
   Compiling wurtle v0.1.0 (/workspaces/wurtle)
    Finished dev [unoptimized + debuginfo] target(s) in 1.07s
     Running `target/debug/wurtle`
Hello, world!
```

Let's add Rocket (and its JSON feature, which we'll need for parsing) to our dependencies in `Cargo.toml`:

```plaintext
[dependencies]
rocket = { version = "0.5.0-rc.1", features = ["json"] }
```

Now, we can set up a barebones Rocket server in our `main.rs`:

```rust
#[macro_use] extern crate rocket;

#[get("/")]
fn handler() -> &'static str {
    "Hello world!"
}

#[launch]
fn rocket() -> _ {
    rocket::build().mount("/", routes![handler])
}
```

Run this minimal example with `cargo run` and then open `http://localhost:8000/` in your browser, and you'll see that static string response: `Hello world!`

But we want to send (and receive) JSON objects. Rocket uses the serde crate internally for JSON serialization and deserialization, so we can define the format of our request and response objects and tell Rocket to parse them for us:

```rust
//....
use rocket::serde::{Serialize, Deserialize, json::Json};

// GuessRequest will be deserialized from JSON to a Rust struct
#[derive(Deserialize)]
#[serde(crate = "rocket::serde")]
struct GuessRequest<'r> {
    guess: &'r str,
    word: Option<usize>, // `word` is optional
}

// GuessResponse will be serialized from a Rust struct to JSON
#[derive(Serialize)]
#[serde(crate = "rocket::serde")]
struct GuessResponse<'a> {
    result: Vec<&'a str>,
    word: usize,
    win: bool,
}

// The POST request's data will be read in JSON format and passed to the `message`
// parameter of the handler.
#[post("/guess", format = "json", data = "<message>")]
fn handler(message: Json<GuessRequest<'_>>) -> Json<GuessResponse<'_>> {
    Json(GuessResponse {
        result: vec!["CORRECT", "CORRECT", "ALMOST", "WRONG", "WRONG"],
        word: message.word.unwrap_or(0),
        win: false
    })
}
//....
```

Let's take a brief diversion on lifetimes. If you're not familiar with Rust syntax, the `<'_>` part may be baffling. This is a way to tell Rust when it can clean up variables: on our GuessRequest struct, for example, the lifetime of the struct is `'r`. The lifetime of the `guess` property is the same. So the `guess` property needs to be kept around as long as the struct is around. The `'_` lifetime on the `hello` method's types tells Rust that the Request and Response structs will have the same lifetime and can be discarded when both are no longer needed.

Lifetimes are one of the more difficult concepts to grasp, so if you'd like to dig deeper, check out [the explanation in the docs](https://doc.rust-lang.org/rust-by-example/scope/lifetime.html).

That done, we now have an API - sort of. We can post a JSON object to `/guess`, and we'll receive a JSON response. But we aren't actually checking the word against anything. Let's create a new file, `words.rs`, and add some logic.

```rust
use rand::{thread_rng, Rng};

static LINES: &str = include_str!("words.txt");

pub fn get_word(index: usize) -> Option<&'static str> {
    LINES.lines().nth(index)
}

pub fn is_valid_word(word: &str) -> bool {
    LINES.lines().any(|dict_word| dict_word == word)
}

pub fn get_random_word() -> usize {
    let mut rng = thread_rng();
    rng.gen_range(0..LINES.lines().count())
}
```

Here we're creating a static variable, LINES. At compile time, cargo will read "words.txt" from our `src/` directory and include the contents in our compiled binary. We could also package the file with the binary and read it at runtime, but this is faster and will work fine for our purposes. (If you're following along, go ahead and create a `words.txt` file and add a handful of random five-letter words.)

Then we have some helper functions: we can look up words in the list by index; we can check to see if the word is valid (instead of just random characters); and we can get a random index of a word from the list. (We'll use the index as the word's id in our requests & responses.)

We'll need to add the `rand` dependency to our Cargo.toml:

```plaintext
[dependencies]
rand = "0.8.5"
```

Then let's import this module in our main file and improve our `guess/` method:

```rust
mod words;

//....

#[post("/guess", format = "json", data = "<message>")]
fn handler(message: Json<GuessRequest<'_>>) -> Json<GuessResponse<'_>> {
    let word = message.word.unwrap_or_else(words::get_random_word);
    Json(GuessResponse {
        result: vec!["CORRECT", "CORRECT", "ALMOST", "WRONG", "WRONG"],
        word,
        win: false
    })
}

//....
```

Now, if the optional parameter `word` exists in the request message, we'll use that; otherwise, we'll return a random index from our `words.txt` file.

But we should actually check if the guess matches the target word! Let's create a new module, `guess.rs`, for this logic. We'll start with a function to compare two words.

Note the lifetimes reappearing again; in this case, the output list of `str`s won't reference our input variables at all, so they can have separate lifetimes.

This function's return type is a Result, which means it will either be `Ok()` with a vector of strings or `Err()` with a String error message.

```rust
fn compare_words<'a,'b>(actual: &'a str, guess: &'a str) -> Result<Vec<&'b str>, String> {
    if actual.len() != guess.len() {
        return Err(format!("Guess should have {} letters (was {})", actual.len(), guess));
    }
    // word_chars is mutable because the logic below replaces any matched characters
    // with `_`. guess_chars is never changed.
    let mut word_chars: Vec<char> = actual.to_lowercase().chars().collect();
    let guess_chars: Vec<char> = guess.to_lowercase().chars().collect();
    let mut result = vec![];

    // 'letters is a label for this loop, so we can `continue` it
    // from inside the nested loop below.
    'letters: for index in 0..word_chars.len() {
        let guessed = guess_chars[index];
        let actual = word_chars[index];

        if guessed == actual {
            // Guessed letter matches the word!
            result.push("CORRECT");
            // Replace the letter in the target word
            // so it can't be matched by an out-of-place letter
            word_chars[index] = '_';
        } else {
            // Check if guessed letter has a match in another position
            // in the target word, but ONLY if that letter isn't correctly
            // matched, and ONLY if the out-of-position match hasn't
            // already been matched to another guessed letter
            for actual_char_index in 0..word_chars.len() {
                if word_chars[actual_char_index] != guess_chars[actual_char_index] && word_chars[actual_char_index] == guessed {
                    // Letter exists in a different position in the word
                    result.push("ALMOST");
                    // Replace the letter in the target word
                    // so it can't be matched by an out-of-place letter
                    word_chars[actual_char_index] = '_';
                    continue 'letters;
                }
            }
            // The letter doesn't exist in the target word
            result.push("WRONG");
        }
    }

    Ok(result)
}
```

This logic looks correct, but let's write some test cases to make sure. Rust makes this easy: you can add them in a separate file, or just include them in a `test` module inside this one. Then run `cargo test` to find and execute the test cases. Here's one to get you started:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_win() -> Result<(), String> {
        let response = compare_words("which", "which")?;
        
        assert_eq!(response, vec!["CORRECT", "CORRECT", "CORRECT", "CORRECT", "CORRECT",]);
        Ok(())
    }
}
```

The `?` after `compare_words` will short-circuit the function: if `compare_words` returns an `Err()`, the test will immediately return that `Err()` and end. Otherwise, `response` will be the `Ok()` value.

Try writing some other test cases to check failing conditions. What happens if one word is in UPPERCASE?

Once we're satisfied that the logic for this function works, we'll add in our wordlist from the `words` module:

```rust
use crate::words::{get_word, is_valid_word};

pub fn check_guess<'a,'b>(guess: &'a str, word_index: usize) -> Result<(Vec<&'b str>, bool), String> {
    // If the word is not a real word, return an error
    if !is_valid_word(guess) {
        return Err("Guess is not a valid word".to_string());
    }
    match get_word(word_index) {
        Some(word) => {
            let comparison = compare_words(word, guess)?;
            let win = comparison.iter().all(|r| *r == "CORRECT");

            Ok((comparison, win))
        },
        None => Err("Word not found".to_string())
    }
}
```

`match` is a helpful way to return something different depending on a value. In this case, `get_word` returns an Option, which may be either `Some()` or `None`. If the word index isn't valid, we'll get a `None`, and return an `Err()`.

If it is valid, we'll get a `Some()`, so we'll assign the value to `word` with `Some(word)` and run the block. We'll invoke our `compare_words` function to get the list of matching characters; then, we'll check to see if all of them are "CORRECT". If they are, you win! Then we return both the list of matching characters and the win boolean as a tuple.

Let's add this to our request handler:

```rust
mod guess;

//....

#[post("/guess", format = "json", data = "<message>")]
fn handler(message: Json<GuessRequest<'_>>) -> Json<GuessResponse<'_>> {
    let word = message.word.unwrap_or_else(words::get_random_word);
    let (result, win) = guess::check_guess(message.guess, word).map_err(|err| status::BadRequest(Some(err)))?;
    Json(GuessResponse {
        result,
        word,
        win
    })
}

//....
```

`check_guess` can return an `Err()` with a `String` inside, but we'd really like to specify that this should return a `400 Bad Request` status code if there's an error. So, we'll map that error, returning a `status::BadRequest` with the error message instead. Then, again, the `?` will short-circuit the function if there's an error, returning that BadRequest. Rocket will generate the response and set the status code for us.

If there's no error, then we'll just create the GuessResponse struct with our `result`, `word`, and `win` variables.

And that's it! We now have a minimal, but functional, word game API.

![Example response from the API](assets/rust-api-response.png)

## In Review

So is Rust the ideal language for general API development? Perhaps not. The details of borrow-checking, lifetimes, etc. may be unnecessary overhead for developers. But Rust excels at safe, memory- and CPU-efficient code. If performance is a concern - say, for high-volume data processing pipelines - Rust may be a good candidate. Development is getting ever easier with the growing ecosystem of third-party crates like Rocket.

Now is a great time to consider: Where does Rust fit into your future application development efforts?