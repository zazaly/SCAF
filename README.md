# SCAF
We are designing a new digital comic book file format called ["Seekable Comic Archive Format (SCAF)"]. Based on Zstandard's seekable format principles: independent compressed frames + a seek table at the end (in a skippable frame) that enables random access to pages, panels, or specific assets without decompressing the entire file.
