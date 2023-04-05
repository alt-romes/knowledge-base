#program-verification

We only assure properties if the program terminates

Separation logic for heap-manipulating programs, Hoare logic more for imperative
programs?

An imperative program is a state transformer

An assertation is a \textbf{pure observation} of the state of the program

Hints for finding loop invariants

- Carefully think about the post condition of the loop
	 - Typically the post-condition talks about a property acuumulated across a range
	 - e.g. maximum of all elements of an array
	 - e.g. sort visited elements in a data structure
- Design a generalized version of the post condition
	 - Break the invariants within the loop as long as they are restored