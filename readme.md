## Parsing

As with most parsing approaches, we're parsing the input strings in two main steps; the tokenizer and the lexier; with input validation before anything.

### Parser input validation
Before any of the parsing processes take place, the code is doing a simple grammar check which validates the user input string. We do this by using a simple set of rules. For each token, we defined which tokens are allowed to be placed before the token, and which tokens are allowed to be placed after it. This is heaven for playing around with bits, and is done by assigning each token type a unique bit and using bitmasks. When the current token is not suitable with the previous token or the next token, an error is thrown. Note that all spaces are skipped in this phase, since they don't affect the validation of the program.

The method above fails to account for errors in tokens which have a broader scope. For example, a quote could be the first token encountered, but closed at some unknown location further on in the string - and the same rules for brackets. In both cases, the tokens are checked using the method described above, but an additional step is taken.

- Quotes: A flag is set once the checker sees a quote, and is unset once the checker sees the closing quote. When at the end of the string this flag is set, an error is thrown.
- Brackets: A counter is set at the beginning of the checker. Every time it encounters an opening bracket, the counter is increased, and every time it encounters a closing bracket, the counter is decreased. If, at any point in the program the number becomes negative, an error is thrown. And if this counter is not equal to 0 at the end of the string, another error is thrown.

After these checks, we're ready to get the parsing started! Note that from now on, our parsing system will assume the data is correct and will not do any additional checks on the input.

### Tokenizer
At first, the **full** input string is being tokenized, returning a linked list of tokens and characters.

```c
char    *input_string = "ls >>o"

t_list  *tokenizer_output = tokenizer(input_string);
/* tokenizer_output:
node [0] = char 'l'
node [1] = char 's'
node [2] = split <token>
node [3] = output_append <token>
node [4] = char 'o'
node [5] = NULL */
```

The tokenizer recognizes quotes, and will parse all special tokens within them as normal strings. For example; a normal space is parsed as the `split token`, but within quotes the `char ' '` representation is used.

!! There is one exception to the rule here. Whenever the code comes across the HEREDOC token, it will handle it accordingly, and store the used filedescriptor as data payload with the token. The string representing the closing state of the heredoc will not show up in the tokenizer output. For example:

```c
char    *input_string = "cat <<EOF"

t_list  *tokenizer_output = tokenizer(input_string);
/* tokenizer_output:
node [0] = char 'c'
node [1] = char 'a'
node [2] = char 't'
node [3] = split <token>
node [4] = input_heredoc <token> fd: 3
node [5] = NULL */
```

And then, we have the case of brackets. Just like the HEREDOC token, the OPEN_BRACKET token has some additional payload: the tokenized list of everything inside the brackets. Note that the closing bracket will not be stored, but "recognized" with the end of the child list. For example:

```c
char    *input_string = "ls&& (ls ) "

t_list  *tokenizer_output = tokenizer(input_string);
/* tokenizer_output:
node [0] = char 'l'
node [1] = char 's'
node [2] = AND <token>
node [3] = split <token>
node [4] = open_bracket <token> list:
    child_node [0] = char 'l'
    child_node [1] = char 's'
    child_node [2] = split <token>
    child_node [3] = NULL
node [5] = split <token>
node [6] = NULL */
```

> To save space, tokenizer output will be shown as a single line. It's still represented as a linked list under the hood though!

### Lexier (basics)
After we get the tokenized linked list, it's time to get some execution done. Let's assume for now that our input is just a single command like `echo hello | grep h >$OUTFILE`.

The first step when encountering a command is expanding values. Our shell is handling two: environment variables and wildcards.

- In the case of an environment variable, the name of the variable is stored and deleted from the tokenizer list. A value for the variable is searched and added to the tokenizer list if found. Spaces are handled as tokens, unless they're within a quote.
- In the case of a wildcard: the wildcard pattern is fetched from the tokenizer list, and handled. If no results are found, the wildcard tokens within the pattern are changed into normal strings (`wildcard <token> --> char '*'`). If at least one match is found, the full pattern will be removed from the tokenizer list, and the results are pushed to the tokenizer list; seperated with a space. Obviously, following the same rule with quotes and spaces.

After this stage, our input will look like this: `echo hello | grep h >out.txt`. From here, it's as easy as going through all the tokens. If the token you encounter is a char, you take the next full string and add it to the argument list. If the token you encounter is a special token, handle it accordingly. So in the case of an input/output: fetch the next string and open the file while storing the file descriptor. In the case of the input heredoc, load the file descriptor gained in the tokenizer, and in the case of the pipe, create a new program instance to be executed.

### Lexier (advanced, execution loops)
The reality is though, assuming the input is just a single command ain't aligning with reality. In between the normal commands described above, 3 different operators are implemented:

- && (AND operator). The next command will **only** execute when the previous command exits successfully (exit status == 0).
- || (OR operator). The next command will **only** execute when the previous command exited unsuccessfully (exit status != 0)
- ; (DO operator). The next command will be executed, no matter what.

In between the tokenizer and the Lexier, we have an interpreter. Its default mode is DO - execute the next command you find. So as soon as it comes across a command, it will call the Lexier and execute it accordingly. Since the lexier will stop at the first token that's not describing a command (after executing!), it will return to the interpreter.

For example: (Note that -> is not a part of the command)

```
-> echo 'Hello world!' | grep '!' && echo 'Someone is using a !' || echo 'Someone is boring'
The interpreter comes across the first token `e`, and sees that it is part of a single command. Because the default mode is DO - the command is parsed and executed.

-> && echo 'Someone is using a !' || echo 'Someone is boring'
After execution, the next token is `&&`. The interpreter changes mode to AND, and continues. The next token (`e`) is once again part of an command. Because the return value of the previous command was 0 (success), and the mode is AND, the command should be parsed and executed.

-> || echo 'Someone is boring'
After execution, the next token is `||`. The interpreter changes mode to OR, and continues. This time, it sees another command but wont execute it. Instead, it's skipping over all tokens within the command; and will continue afterwards.

Since nothing is left in the string, the interpreter returns.
```

##

Project made together with [Jared Goedhart](https://github.com/jaredgoedhart), as part of the 42 curriculum at the Codam Coding College in Amsterdam.