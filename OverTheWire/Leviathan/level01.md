# Leviathan Level 1 → 2

## What I did

After logging in as `leviathan1`, I checked the home directory.

```bash
ls -la
```

I found an executable called `check`.

I wanted to know what the program was doing, so instead of randomly guessing the password, I decided to use `ltrace`.

```bash
ltrace ./check
```

The program asked me to enter a password.

After entering an input, I looked at the output from `ltrace`. I noticed that the program was calling `strcmp()` to compare strings.

This was interesting because `strcmp()` is a C function used to compare two strings.

I learned that it returns `0` when both strings are equal.

By looking at the arguments passed to `strcmp()` in the trace, I could figure out what password the program expected.

After entering the correct password, the program gave me a shell.

Then I read the password for the next level:

```bash
cat /etc/leviathan_pass/leviathan2
```

## What I learned

- `ltrace` can show library function calls made by a program.
- `strcmp()` is commonly used for password comparison.
- I don't always need to reverse engineer the entire binary to understand a simple password check.
- Dynamic analysis can save a lot of time when investigating programs.

## Final thoughts

This was my first time using `ltrace` to investigate a binary.

I initially thought I would need to guess the password, but tracing the function calls made the process much easier.

It also made me curious about how programs handle user input and password checking internally.

