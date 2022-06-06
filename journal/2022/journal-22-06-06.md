# Levo Journal Entry #1: Lexical analysis — Part 1 #
June 6-12, 2022<br>
by Ahraman

## Intro ##

Hi, and welcome to the first journal entry about the evolution, design and implementation of the Levo programming language, together with its compiler. The goal with
Levo is to create a high-level statically-typed object-oriented programming language which looks the way I would like a programming langauge to look, and the supports
as many features I can implement for it. It should have an ergonomic, easy and intuitive syntax but must also be capable of doing systems programming.

We will return to the problem of language design later, but for the time being, we will focus on the first stage, which is to a large extent independent of particular
language-dependent decisions.

The goal of this entry is to document the creation of a basic lexer—a lexical analyzer—which is tasked with converting source text into a series of identifiable tokens
by the parser.

## Preliminaries ##

The code for this version of Levo compiler will be written in the C programming language. Furthermore, until a further notice, the main target OS will be Windows;
there is no particular reason for this decision under than it being the OS that I mostly work on. In the future, I will consider porting the code into other OSes as
well.

### Source texts ###

The first step is loading source files into the memory for processing. The data is stored in the following opaque struct:
```c
typedef struct levo_source* levo_source_t;
```
The content of this struct is as follows:
```c
struct levo_source {
    levo_character_encoding encoding; // The encoding of the source.
    unsigned char* cur; // The current position within the buffer. This is used by reading the files.
    unsigned char* start; // The start of the buffer containing the source text. It is the start of a null-terminated string.
    unsigned char* end; // The end of the buffer containing the source text. It points to the null character at the end.

    unsigned char* to_free; // The pointer to free when destroying this struct.
};
```
The ```levo_character_encoding``` enum type represents the encodings currently supported, which are meager at the moment:
```c
typedef enum levo_character_encoding {
    LEVO_CHARACTER_ENCODING_UNKNOWN = 0,
    LEVO_CHARACTER_ENCODING_UTF8 = 1,
} levo_character_encoding;
```

The following function loads a file at ```path``` and reads the content into a created ```levo_source_t```.
```c
levo_result levo_source_from_file(
    levo_source_t* source_ptr,
    const char* path,
    levo_character_encoding encoding) {
    // arg validity checks
    if (!source_ptr || !path) {
        return LEVO_ERROR_NULL_ARGUMENT;
    }
    if (encoding == LEVO_CHARACTER_ENCODING_UNKNOWN) {
        // a valid encoding must be provided
        return LEVO_ERROR_INVALID_ARGUMENT;
    }

    // allocate struct
    if (!(*source_ptr == (levo_source_t)levo_alloc(sizeof(struct levo_source)))) {
        // allocation failed
        fprintf(stderr, "%s\n", "error: allocation failed");
        return LEVO_ERROR_ALLOCATION_FAILED;
    }

    levo_result result = LEVO_SUCCESS;

    // open file and read content
    size_t content_size;
    unsigned char* content;
    if (result = levo_read_file(path, &content_size, &content)) {
        goto on_error;
    }

    // initialize struct
    (*source_ptr)->encoding = encoding;
    (*source_ptr)->start = content;
    (*source_ptr)->end = &content[content_size];
    (*source_ptr)->cur = (*source_ptr)->start;
    (*source_ptr)->to_free = content;

    return LEVO_SUCCESS;
on_error:
    levo_free(*source_ptr);
    return result;
}
```
The enum ```levo_result``` is a type representing various different result values, the most common one being ```VK_SUCCESS``` with value ```0```, which indicates
success; furthermore, the ```encoding``` provided is used for reading the content. Within the code, the function ```levo_result levo_read_file(const char*, size_t*,
unsigned char**)``` opens a file and reads the content and its size as output, which I omit for the sake of brevity.

For cleanup, I use the following function:
```c
void levo_destroy_source(
    levo_source_t source) {
    if (source) {
        levo_free(source->to_free);
        levo_free(source);
    }
}
```

The functions ```void* levo_alloc(size_t)``` and ```void levo_free(void*)``` are currently aliases for `malloc` and `free` functions of C standard library,
respectively.

### Tokens ###

To simplify matters for the time being, I assume that ALL text is in UTF-8 encoding. This allows us to take each individual byte in the buffer to represent a UTF-8
char. Thus, to represent a lexical token, I use the following struct:
```c
typedef struct levo_token {
    levo_token_type type;   // The type of the token.
    levo_source_t source;   // The source file in which this token appeared.
    unsigned char* begin;   // Beginning of the token in source.
    unsigned char* end;     // End of the token in the source. The length of the token is end - begin.
} levo_token;
```
Note that since tokens refer to the source buffer, this means that ```levo_source_t``` instances must be kept in memory until the program ends.

We will try to limit the types of tokens possible in this toy example. Therefore, we will have the following list of token types:
```c
typedef enum levo_token_type {
    LEVO_TOKEN_TYPE_UNKNOWN = 0,

    LEVO_TOKEN_TYPE_IDENTIFIER = 1,         // ident
    LEVO_TOKEN_TYPE_INT_LITERAL = 2,        // int_lit

    LEVO_TOKEN_TYPE_LEFT_PARENTHESIS = 3,   // (
    LEVO_TOKEN_TYPE_RIGHT_PARENTHESIS = 4,  // )
    LEVO_TOKEN_TYPE_PLUS = 5,               // +
    LEVO_TOKEN_TYPE_MINUS = 6,              // -
    LEVO_TOKEN_TYPE_STAR = 7,               // *
    LEVO_TOKEN_TYPE_SLASH = 8,              // /
    LEVO_TOKEN_TYPE_EQUALS = 9,             // =

    LEVO_TOKEN_END_OF_FILE = -1,
} levo_token_type;
```

The function responsible for extracting a token from the source is as follows:
```c
levo_result levo_next_token(
    levo_source_t source,
    levo_token* token);
```
We shall return to implementing this function later.

### Unicode characters ###

With the source codes now loaded, we must implement methods to access the characters individually. The following function will peek the next character in the source
file:
```c
size_t levo_peek_char_from_source(
    levo_source_t source,
    uint32_t* c) {
    // args validity check
    if (!source || !c) {
        return 0;
    }

    // Code differs based on encoding
    switch (source->encoding) {
    case LEVO_CHARACTER_ENCODING_UTF8:
        return levo_next_utf8(source->cur, source->end, c);
    case LEVO_SOURCE_ENCODING_UNKNOWN:
        // not supported.
        return 0;
    }
}
```
This function calls other ones based on the encoding and writes the output character to ```c```, and returns the length of the character in bytes based on said
encoding. For _reading_ the character—that is, advancing ```source->buf``` as well, the following function is used instead:
```c
size_t levo_next_char_from_source(
    levo_source_t source,
    uint32_t* c)
{
    size_t adv = levo_peek_char_from_source(source, c);
    if ((size > 0) && (*c != -1)) {
        source->cur += adv;
    }
    
    return adv;
}
```

The function for UTF-8 encoding is defined as follows:
```c
size_t levo_peek_utf8(
    const unsigned char* cur,
    const unsigned char* end,
    uint32_t* c) {
    if (cur >= end) {
        *c = -1;
        return -1;
    }

    uint32_t c0 = cur[0];
    if (cur[0] & 0x80) { // 1-byte
        *c = c0;
        return 1;
    }
    else if ((cur[0] & 0xE0) == 0xC0) { // 2-byte
        if (cur + 1 >= end) {
            *c = -1;
            return 2;
        }

        uint32_t c1 = cur[1];
        *c = ((c0 & 0x1F) << 6) | (c1 & 0x3F);
        return 2;
    }
    else if ((cur[0] & 0xF0) == 0xE0) { // 3-byte
        if (cur + 2 >= end) {
            *c = -1;
            return 3;
        }

        uint32_t c1 = cur[1];
        uint32_t c2 = cur[2];
        *c = ((c0 & 0x0F) << 12) | ((c1 & 0x3F) << 6) | (c2 & 0x3F);
        return 3;
    }
    else if ((cur[0] & 0xF8) == 0xF0) { // 4-byte
        if (cur + 3 >= end) {
            *c = -1;
            return 4;
        }

        uint32_t c1 = cur[1];
        uint32_t c2 = cur[2];
        uint32_t c2 = cur[3];
        *c = ((c0 & 0x07) << 18) | ((c1 & 0x3F) << 12) | ((c2 & 0x3F) << 6) | (c3 & 0x3F);
        return 4;
    }
    else { // invalid
        *c = cur[0];
        return 0;
    }
}
```

Disclaimer that at this stage, this is likely not the most efficient implmenetation of the function. We will revisit character encodings later on.

## Lexical analysis ##

We'll begin with describing a 
