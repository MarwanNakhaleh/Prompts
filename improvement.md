# Read books
--------
/clear Hello Claude, we need to improve our prompts with ideas we've learned from our studies. Please read through all prompts in @ios/ and @web/, then search for a file called reading_progress.json. With the remaining context, start reading "current_book" beginning at position "current_position" (line number) from the JSON file. If there is context left over, begin reading "next_book". Determine what improvements we've learned from the material you've read, then modify the prompts in those folders to reflect your improved knowledge. When you are finished with this exercise, update reading_progress.json. If we had to read "next_book", set current_book to the existing value of next_book, then set next_book's value to "". Set "current_position" to the position in the book we reached with this context window. Do not make references to the original text files in the updated prompts, instead either capture the concepts as much as is needed for an LLM within the prompt, or point it to search the web for expanded ideas. If anything has to change in README.md to support these changes, please update that file as well. I approve all reads and edits for this session. Thank you
--------
/clear Hello Claude, we need to improve our prompts with ideas we've learned from our studies. Please read through all prompts in @ios/ and @web/, then search for a file called reading_progress.json.

CRITICAL FIRST STEP: Run `ls knowledge-base/` to discover all available books before reading reading_progress.json. Use this listing to determine what "next_book" should be set to when the current book finishes. Never assume the book order from memory — always derive it from the directory listing.

With the remaining context, start reading "current_book" beginning at position "current_position" (line number) from the JSON file. Read in chunks of ~1500 lines to avoid hitting context limits. If the remaining content of a book appears to be appendix material, raw code listings, or bibliography rather than conceptual content, it is acceptable to skip to the end, mark the book complete, and begin "next_book" instead of reading through it.

If there is context left over after finishing "current_book", begin reading "next_book". Determine what improvements we've learned from the material you've read, then modify the prompts in those folders to reflect your improved knowledge. When applying insights:
- Cross-reference against what is already in the prompts before adding — only add what is genuinely net-new
- Prefer deepening an existing principle over adding a new section when the concept is closely related
- Capture concepts inline in the prompt language; do not reference source text files by name

When you are finished with this exercise, update reading_progress.json:
- Set "current_position" to the line number reached in the current book
- If a book is fully complete, promote: current_book ← next_book, then set next_book to the following entry from the `ls knowledge-base/` listing
- If "next_book" was also started this session, set next_book to the entry after that

Do not make references to the original text files in the updated prompts, instead either capture the concepts as much as is needed for an LLM within the prompt, or point it to search the web for expanded ideas. If anything has to change in README.md to support these changes, please update that file as well.

Once you have completed this step, compact your context and repeat this process until all books are processed and completed. So far, the following books have been fully processed:

- knowledge-base/agile-testing.txt
- knowledge-base/building-microservices.txt
- knowledge-base/clean-architecture.txt
- knowledge-base/clean-code.txt
- knowledge-base/dsa-cplusplus.txt
- knowledge-base/effective-unit-testing.txt
- knowledge-base/enterprise-integration-patterns.txt
- knowledge-base/growing-object-oriented-software.txt
- knowledge-base/continuous-delivery.txt 
- knowledge-base/implementation-patterns.txt
- knowledge-base/intro-to-algorithms.txt
- knowledge-base/designing-interfaces.txt 
- knowledge-base/object-oriented-design-patterns.txt
- knowledge-base/designing-data-intensive-applications.txt
- knowledge-base/domain-driven-refactoring.txt

I approve all reads and edits for this session. Thank you

--------
