# optimizebuffer
ROOT OptimizeBuffer function analysis and changes

``void TTree::OptimizeBaskets(ULong64_t maxMemory, Float_t minComp, Option_t *option)``

Takes in some maxMemory, minimum compression factor, and a debug flag


1. **Calculates** the best values for branch buffer sizes s.t. the total buffer size is less than maxMemory
2. **If** the branch compression factor for the data written so far is less than the set minComp **then** compression is disabled. 

--- 

### 1. Calculates best buffer sizes: 

**If** the file is writable, **then** flush the existing baskets. 
`if (this->GetDirectory()->IsWritable()) this->FlushBasketsImpl();`

```
TObjArray *leaves = this->GetListOfLeaves();
Int_t nleaves = leaves->GetEntries();

Double_t treeSize = (Double_t)this->GetTotBytes();

if (nleaves == 0 || treeSize == 0) {
	return;
}
```
Decides if the function is too early to be called.

--- 

#### Some initialization
```
Double_t aveSize = treeSize/nleaves;
UInt_t bmin = 512; //512 kB
UInt_t bmax = 256000; //25.6 MB

Int_t i, oldMemsize,newMemsize,oldBaskets,newBaskets;
i = oldMemsize = newMemsize = oldBaskets = newBaskets = 0;
```

---

#### Start pass (1/2)

At the start of the pass, we'll establish 
(**oldMemsize**), (**newMemsize**), (**oldBaskets**), (**newBaskets**):

```
oldMemsize = 0; //to count size of baskets in memory with old buffer size

newMemsize = 0; //to count size of baskets in memory with new buffer size

oldBaskets = 0; //to count number of baskets with old buffer size

newBaskets = 0; //to count number of baskets with new buffer size
``` 
These will be updated as we continue in this pass

> Loop through `nleaves` 

At the start of every loop, establish: 
```
TLeaf *leaf = (TLeaf*)leaves->At(i); // at leaf i

TBranch *branch = leaf->GetBranch(); // get its branch

Double_t totBytes = (Double_t)branch->GetTotBytes(); // how big is it

Double_t idealFactor = totBytes/aveSize; //briefly calculate its ideal comp factor

UInt_t sizeOfOneEntry; //init
```

```
if (branch->GetEntries() == 0) {

// There is no data, so let's make a guess ...
sizeOfOneEntry = aveSize;

} else {

sizeOfOneEntry = 1+(UInt_t)(totBytes / (Double_t)branch->GetEntries());

}
```

So $$\text{Size of One Entry} = 1 + \frac{\text{Total Bytes}}{branch(i)\rightarrow\text{GetEntries()}}.$$ #HELP Why are we doing this here?
- If this leaf `i`'s branch has no entries, then set the size of one entry to the average size, 
- else we can actually calculate the size of a single entry by taking the number of entries the branch has, dividing it from the total number of bytes the branch has and adding one to the quotient. 
- #TRY calculating this for an example previously worked with!

---

Next we'll set up some temp vars and update (**oldMemsize**) and 
(**oldBaskets**)
```
Int_t oldBsize = branch->GetBasketSize();//est oldBsize
oldMemsize += oldBsize; // use oldBsize to update oldMemsize from 0
oldBaskets += 1+Int_t(totBytes/oldBsize); //est oldBaskets
```

Here we'll establish the **n**umber of **b**ranches (**nb**): 
```
Int_t nb = branch->GetListOfBranches()->GetEntries();
```

Now we'll update the (**newBaskets**), if (**nb**) is not 0, then 

```
if (nb > 0) {
	newBaskets += 1+Int_t(totBytes/oldBsize);
	continue; //loop to the next leaf, i++;
}
```
in other words, if (**nb**) > 0, then change (**newBaskets**) and keep (**oldBaskets**) the same and iterate to the next "leaf". 


if (**nb**) = 0, then continue on to the next steps with the current leaf.

#HELP I'm not sure how this is supposed to accomplish what we're setting out to do. 
- Why does the number of branches dictate whether (**newBaskets**) gets updated? 

---

In any case, with current leaf `i` in hand, set a new 
``bsize = oldBsize*idealFactor*memFactor;`` which can be large. 

```
if (bsize < 0) bsize = bmax;
if (bsize > bmax) bsize = bmax;
```

#HELP I see how this sets a bsize that's too large to something preset (25.6 MB), but why would we set a negative `bsize` to `bmax` also?
- Why not set it to `bmin`? 
- When would it have the chance to be negative? It shouldn't because it relies on the number of entries in each branch. 
- If this never happens, is it just a catch that'll help during debugging?

---

In any case, set `newBsize = bSize`,  which, if it's less than the `sizeOfOneEntry`, make the `newBsize = sizeOfOneEntry`

```
if (newBsize < sizeOfOneEntry) newBsize = sizeOfOneEntry;
```
and if that's still less than `bmin` , make `newBsize = bmin`
```
if (newBsize < bmin) newBsize = bmin;
```


If neither of those are the case, and `newBsize = bsize > 10 million bytes = 10 MB`
```
if (newBsize > 10000000) newBsize = bmax;
```

Then add `newBsize` to current `newMemsize`  

> Keep looping through all the leaves before the 2nd pass. Updating `newBsize` and `newMemsize` for every leaf. 


---

#### Start Pass (2/2)

On the second pass, we're updating `newBsize`, `oldMemsize`,  and `oldBsize` with each leaf, same as before, until we hit the first if statement ``if(pass)``

```
if(pass) {
	Long64_t clusterSize = (fAutoFlush > 0) ? fAutoFlush : branch->GetEntries();
	if (branch->GetEntryOffsetLen()) {
		newBsize = newBsize + (clusterSize * sizeof(Int_t) * 2);
	}
	.
	.
	.
}
```
this part is accounting for an entry offset for each leaf by first establishing a `clusterSize`, which is either going to be:
1. `fAutoFlush`, which here will be a set value of the number of entries`TTree:Fill` will flush all branch buffers to disk. 
2. `branch->GetEntries()` if `fAutoFlush` isn't already setup. This serves as a catch which will set the cluster to the whole number of entries in the branch so we over estimate instead of underestimate. 

Then if we do have a an entry offset for the branch, we'll adjust the new basket size accordingly. If not, then there's no need to have done any of that, and we keep `newBsize` unchanged. 

Regardless of the previous change, the second part of this first `if(pass)` statement will adjust `newBsize` 

```
if(pass){
	.
	.
	.
	newBsize = newBsize  - newBsize%512 + 512; 
}
```

#HELP I'm not sure what this is trying to accomplish. 
	It shifts the new buffer size down by (however many times the new buffer can hold 512 bytes) and then shifts that up by one. But why? Is this setting a minimum buffer size to `newBsize + 512 bytes`? Or is this making `newBsize` divisible by 512? (It seems like both, **but why is that the intention?**) 

After this line, we have a `newBsize` divisible by 512 bytes (unless we don't), and we're going to set our `newBsize` to  `bmin` and `bmax` accordingly as we did on the first pass. Then, we're going to set the buffer size for the this branch to `newBsize`. 

Following with reupdating `newMemsize` and `newBaskets`  and here is where we'll finish with the last taks this function is set to do.  


---

### 2. Decide compression

```
//Reset the compression level in case the compression factor is small

Double_t comp = 1;
if (branch->GetZipBytes() > 0) comp = totBytes/Double_t(branch->GetZipBytes());
if (comp > 1 && comp < minComp) {
	if (pDebug) Info("OptimizeBaskets", "Disabling compression ...");
	branch->SetCompressionSettings(...);
}
```

This establishes a base compression factor of 1, and resets it according to it's ratio of total bytes to zip bytes. $$\text{ Compression Factor } = \frac{\text{Total Bytes}}{branch(i) \rightarrow \text{GetZipBytes()}}.$$

What it'll then decide is, if it's greater than 1 and less than the minimum compression factor we'd be after by default(?)

#HELP I think my thinking here is correct, but I could be wrong. Within the ``branch->SetCompressionSettings(...)`` it's calling ``ROOT::RCompressionSetting::EAlgorithm::kUseGlobal``, which, without looking further into the documentation, looks like it's a separate compression algorithm. Not sure if it's worth investigating into more.

---

## In conclusion

We want to set the absolute maximum to somewhere around the order of 256kB, and while we're already setting that as our `bmax`, it's likely we're multiplying it by 100 at the end of each pass, which will bring the maximum buffer size to 25.6 MB. 

A little more time and maybe clearing up some of the questions I have might be a good step forward. 


