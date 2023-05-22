# Threads

**Threads** are an automation primitive for Solana developers. Just as traditional applications use threads to execute instructions on a computer, smart-contracts can use Clockwork threads to execute a series of instructions on Solana. To create a thread, developers must provided two critical pieces of information:

1. **A trigger** – Some scenario or condition that will gate execution and can verified by a smart-contract.
2. **A set of instructions** – These are instructions that will be run. They can be either statically defined or dynamically built at execution time.
