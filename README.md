# AVCLASS++: Yet Another Massive Malware Labeling Tool

![python](https://img.shields.io/badge/Python-3.6-blue?style=flat-square) [![arsenal](https://img.shields.io/badge/Black%20Hat-Europe%202019%20Arsenal-blue?style=flat-square)](https://www.blackhat.com/eu-19/arsenal/schedule/index.html#avclass-yet-another-massive-malware-labeling-tool-18175)

AVCLASS++ is an appealing complement to [AVCLASS](https://github.com/malicialab/avclass) [1], a state-of-the-art malware labeling tool.

## Overview

AVCLASS++ is a labeling tool for creating a malware dataset. Addressing malware threats requires constant efforts to create and maintain a dataset. Especially, labeling malware samples is a vital part of shepherding a dataset. AVCLASS, a tool developed for this purpose, takes as input VirusTotal reports and returns labels that aggregates scan results of multiple anti-viruses. And now, AVCLASS++ is shipped with the brand-new capacities!

In a nutshell, AVCLASS++ enables the following operation:

- Input:
    - VirusTotal report(s)
    - Malware binar(y|ies) (optional)
- Output:
    - Malware label(s) (family name)

## Features

AVCLASS++ is developed for freeing you from the task of worrying about what families malware samples are. The salient features of AVCLASS++ are as follows:

- *Automatic.* AVCLASS++ removes manual analysis limitations on the size of the input dataset.
- *Vendor-agnostic.* AVCLASS++ operates on the labels of any available set of AV engines, which can vary from sample to sample.
- *Cross-platform.* AVCLASS++ can be used for any platforms supported by AV engines, e.g., Windows or Android malware.
- *Does not require executables.* AV labels can be obtained from online services like VirusTotal using a sample's hash, even when the executable is not available. *Yet, AVCLASS++ has also a potential that can improve label accuracy if there is an executable.*
- *Quantified accuracy.* The original AVCLASS had evaluated [1] on five publicly available malware datasets with ground truth. *AVCLASS++ is further tuned to perform under adverse conditions.*
- *Open source.* We are happy to release AVCLASS++ to the community. Prithee, use it for the further development of prompt security operation and reproducible security research!

## Step Forward

The following limitation was pointed out in the original AVCLASS paper:

> The main limitation of AVClass is that its output depends on the input AV labels. It tries to compensate for the noise on those labels, but cannot identify the family of a sample if AV engines do not provide non-generic family names to that sample. In particular, it cannot label samples if at least 2 AV engines do not agree on a non-generic family name. Results on 8 million samples showed that AVClass could label 81% of the samples. In other words, it could not label 19% of the samples because their labels contained only generic tokens.

We have organized such pitfalls into two factors.

- *First,* AVCLASS is prone to fail labeling samples that have just been posted to VirusTotal because only a few anti-viruses give labels to such samples. Such a sample will be labeled SINGLETON. *An inconvenient truth: when we provided AVCLASS with 20,000 VirusTotal reports, half of them were labeled SINGLETON.*
- *Second,* AVCLASS cannot determine if the label is randomly generated (as with domain generation algorithms of malware) or not. Some anti-viruses that VirusTotal has worked with after AVCLASS released were labeled with the DGA, resulting in a biased label.
 
Because of them, we are forced to make a lot of manual, tedious intervention in malware labeling; otherwise, we need to drop samples with inconsistent labels from the dataset; since there was no alternative.

For the reason, AVCLASS++ is designed to address these drawbacks by arming with the following:

- *Label propagation.* AVCLASS++ accepts not only VirusTotal reports but also binary executable files of samples as input, and measures the similarity between them, thereby propagating [3] a malware label to the one labeled SINGLETON. Here, AVCLASS++ exploits hashed features based on various perspective [4] e.g, byte histogram, printable strings, file size, PE headers, sections, imports, exports, and more! Then it calculates the similarity of the samples through deriving an affinity matrix and re-labels SINGLETONs as a result of the propagation from a similar sample. This enables us to reduce SINGLETONs.
- *DGA detection.* AVCLASS++ determines if  labels were generated by DGA and removes such ones from the candidates. This technique is based on the meaningful characters ratio and $N$-gram normality score [5]. In other words, AVCLASS + + verifies that the label presented by AV is meaningful and easy to pronounce, and then determines if the label is generated by DGA. This enables us to unbiased labeling. 

Besides, unlike AVCLASS, AVCLASS++ is Python 3 compatible!

## How To Use

### Installation

```
git clone git@github.com:malrev/avclassplusplus.git
./setup.sh
```

### Labeling

The labeler takes as input a JSON file with the AV labels of malware samples (`-vt` or `-lb` switches), a file with generic tokens (`-gen` switch), and a file with aliases (`-alias` switch). It outputs the most likely family name for each sample. If you do not provide alias or generic tokens files, the default ones in the *data* folder are used.

```
python avclass_labeler.py -lb data/malheurReference_lb.json -v > malheurReference.labels
```
  
The above command labels the samples whose AV labels are in the *`data/malheurReference_lb.json`* file. It prints the results to stdout, which we redirect to the *`malheurReference.labels`* file. The output looks like this:

```
aca2d12934935b070df8f50e06a20539 adrotator
67d15459e1f85898851148511c86d88d adultbrowser
```

which means sample aca2d12934935b070df8f50e06a20539 is most likely from the *adrotator* family and 67d15459e1f85898851148511c86d88d from the *adultbrowser* family.

The verbose (`-v`) switch makes it output an extra *`malheurReference_lb.verbose`* file with all families extracted for each sample ranked by the number of AV engines that use that family. The file looks like this:

```
aca2d12934935b070df8f50e06a20539  [('adrotator', 8), ('zlob', 2)]
ee90a64fcfaa54a314a7b5bfe9b57357  [('swizzor', 19)]
f465a2c1b852373c72a1ccd161fbe94c  SINGLETON:f465a2c1b852373c72a1ccd161fbe94c
```

which means that for sample aca2d12934935b070df8f50e06a20539 there are 8 AV engines assigning *adrotator* as the family and  another 2 assigning *zlob*. Thus, *adrotator* is the most likely family. On the other hand, for ee90a64fcfaa54a314a7b5bfe9b57357 there are 19 AV engines assigning *swizzor* as family, and no other family was found. The last line means that for sample f465a2c1b852373c72a1ccd161fbe94c no family name was found in the AV labels.  Thus, the sample is placed by himself in a singleton cluster with the name of the cluster being the sample's hash.

Note that the sum of the number of AV engines may not equal the number of AV engines with a label for that sample in the input file because the labels of some AV engines may only include generic tokens that are removed by AVCLASS++. *In such a case, the propagater described later comes to rescue.*

#### Input JSON Format

AVCLASS++ supports two input JSON formats: 

1. VirusTotal JSON reports (*`-vt` file*), where each line in *file* should be the full JSON of a VirusTotal report as fetched through the VirusTotal API.

2. Simplified JSON (*`-lb` file*), where each line in *file* should be a JSON with (at least) these fields: `{md5, sha1, sha256, scan_date, av_labels}`. There is an example of such input file in *`data/malheurReference_lb.json`* This option works well if you want to use label candidates from a source other than VirusTotal or from a self-made engine.

AVCLASS++ can handle multiple input files putting the results in the same output files (if you want results in separate files, process each input file separately).

You can provide the `-vt` and `-lb` input options multiple times.

```
python avclass_labeler.py -vt <file1> -vt <file2> > all.labels
```

```
python avclass_labeler.py -lb <file1> -lb <file2> > all.labels
```

There are also `-vtdir` and `-lbdir` options that can be used to provide an input directory where all files are VT (`-vtdir`) or simplified (`-lbdir`) JSON reports.

```
python avclass_labeler.py -vtdir <directory> > all.labels
```

You can also combine `-vt` with `-vtdir` and `-lb` with `-lbdir`, but you cannot combine input files of different format. Thus, this command works:

```
python avclass_labeler.py -vt <file> -vtdir <directory> > all.labels
```

But, this one throws an error: 

```
python avclass_labeler.py -vt <file1> -lb <file2> > all.labels
```

At this point you have read the most important information on how to use AVCLASS++. The following sections describe optional steps.

#### Label Propagation

When a sample has just been uploaded to VirusTotal, the original AVCLASS often gives you a SINGLETON label because of the lack of AVs signatures. In such a case, we usually try to disassemble and execute the sample, compare the results to past ones, and then give it the appropriate label.

Therefore, We introduce a function that automates this task. AVCLASS++ retrieves and compares byte histogram, printable strings, file size, PE headers, sections, imports, exports, and so on from the given executable files. Then, it gives the label to SINGLETONs from similar samples. An affinity matrix is derived to compute the similarities here. For label propagation, literally the label propagation algorithm [3] is used.

To use this function, run the following command:

```
python avclass_propagator.py -labels <file1> -sampledir <directory> -results <file2>
```

The input file passed with `-labels` must be created in advance by `avclass_labeler.py` in advance. The directory passed with `-sampledir` must contain samples with the hash values contained in the labels file. The option `-results` is optional. By default, the propagator creates `_pr.labels` file based on a `.labels` file passed as an argument. AVCLASS++ overwrites only SINGLETON labels with predicted labels by default. You can overwrite all original labels with predicted labels by enabling the `-force` option. In addition, you can automatically optimize hyperparameter values by enabling `-opt`. 

```
python avclass_propagator.py -labels input.labels -sampledir samples -results output.labels -opt
```

This feature is contrary to the original AVCLASS manner of "does not require executables", but it is really helpful in practice.

#### DGA Detection

AVs such as BitDefender, AegisLab, Emsisoft, eScan, GData, Ad-Aware, MAX, K7Antivirus, K7GW, Cybereason, and Cyren will output pseudo-randomly generated labels in a similar vein as DGA of malware. You can see an example at VirusTotal: [f315be41d9765d69ad60f0b4d29e4300](https://www.virustotal.com/gui/file/d216c14c218251c3e9e25a8041f5011e9fbd9fc6212a9592ba99d8ed6435a535/detection). This leads the original AVCLASS would be confused.

Therefore, we present a function that removes the label that seems to be generated by DGA. To this end, we employ the following heuristics [5]:

- Meaningful characters ratio. This score indicates how many meaningful words within a label (the higher the better). Specifically, we split the label string $p$ into $k$ subwords $|w_i| ≥ 3$, then compute $R(p) = max(\frac{(\sum_{i=1 \in k}) |w_{i}|)}{|p|}$.
-  $N$-gram normality score. This score indicates how many words which are easy to pronounce within a label (the higher the better). Specifically, we compute $N$-gram $t$ of the label string $p$, count the occurrence $count(t)$ in the dictionary, and calculate the average of them. That is, $S_n(p) = \frac{\sum_{n-gram\;t \in p} count(t)}{|p|-n+1}$ where $N$ is given. From our experience, we highly recommend setting $N > 3$.

The key insight of these scores is that the appropriate label contains a string that is meaningful and easy to pronounce. AVCLASS++ calculates a harmonic mean of these scores and determine if the label is generated by DGA based on a threshold. This function is enabled by default now, but you can configure it with:

```
python avclass_labeler.py -vtdir <directory> -dgadetect <dictionary> <n> <threshold> > all.labels
```

An example is below:

```
python avclass_labeler.py -vtdir <directory> -dgadetect data/top10000en.txt 4 2 > all.labels
```

#### Family Ranking

AVCLASS++ has a `-fam` switch to output a file with a ranking of the families assigned to the input samples. 

```
python avclass_labeler.py -lb data/malheurReference_lb.json -v -fam > malheurReference.labels
```

This will produce a file called *`malheurReference_lb.families`* with two columns:

```
virut 441
allaple 301
podnuha 300
```

The file indicates that 441 samples were classified in the virut family, 301 as allaple, and 300 as podnuha.

This switch is very similar to using the following shell command:

```
cut -f 2 malheurReference.labels | sort | uniq -c | sort -nr
```

The main difference is that using the `-fam` switch all SINGLETON samples, i.e., those for which no label was found, are grouped into a fake *SINGLETONS* family, while the shell command would leave each singleton as a separate family.

#### PUP Classification

AVCLASS++ also has a `-pup` switch to classify a sample as Potentially Unwanted Program (PUP) or malware. This classification looks for PUP-related keywords (e.g., pup, pua, unwanted, adware) in the AV labels.

```
python avclass_labeler.py -lb data/malheurReference_lb.json -v -pup > malheurReference.labels
```

With the `-pup` switch the output of the *`malheurReference.labels`* file looks like this:

```
aca2d12934935b070df8f50e06a20539 adrotator 1
67d15459e1f85898851148511c86d88d adultbrowser 0
```

The digit at the end is a Boolean flag that indicates sample aca2d12934935b070df8f50e06a20539 is (likely) PUP, but sample 67d15459e1f85898851148511c86d88d is (likely) not. *This enables us to focus on PUP research [2] or non-PUP research!*

The PUP classification tends to be conservative, i.e., if it says the sample is PUP, it most likely is. But, if it says that it is not PUP, it could still be PUP if the AV labels do not contain PUP-related keywords. Note that it is possible that some samples from a family get the PUP flag while other samples from the same family do not because the PUP-related keywords may not appear in the labels of all samples from the same family. To address this issue, you can combine the `-pup` switch with the `-fam` switch. This combination will add into the families file the classification of the family as malware or PUP, based on a majority vote among the samples in a family.

```
python avclass_labeler.py -lb data/malheurReference_lb.json -v -pup -fam > malheurReference.labels
```

This will produce a file called *malheurReference_lb.families* with five columns:

```
# Family  Total Malware PUP FamType
virut 441 441 0 malware
magiccasino 173 0 173 pup
ejik  168 124 44  malware
```

For virut, the numbers indicate all the 441 virut samples are classified as malware, and thus the last column states that virut is a malware family. For magiccasino, all 173 samples are labeled as PUP, thus the family is PUP. For ejik, out of the 168 samples, 124 are labeled as malware and 44 as PUP, so the family is classified as malware.

#### Ground Truth Evaluation

If you have ground truth for some malware samples, i.e., you know the true family for those samples, you can evaluate the accuracy of the labeling output by AVCLASS++ on those samples with respect to that ground truth. The evaluation metrics used are precision, recall, and F1 measure.

```
python avclass_labeler.py -lb data/malheurReference_lb.json -v -gt data/malheurReference_gt.tsv -eval > data/malheurReference.labels
```

The output includes these lines:

```
Calculating precision and recall
3131 out of 3131
Precision: 90.81  Recall: 93.95 F1-Measure: 92.35
```

The last line corresponds to the accuracy metrics obtained by comparing AVClass results with the provided ground truth.

Each line in the *`data/malheurReference_gt.tsv`* file has two **tab-separated** columns:

```
0058780b175c3ce5e244f595951f611b8a24bee2 CASINO
```

This sample 0058780b175c3ce5e244f595951f611b8a24bee2 is known to be of the *CASINO* family.  Each sample in the input file should also appear in the ground truth file. Note that the particular label assigned to each family does not matter. What matters is that all samples in the same family are assigned the same family name (i.e., the same string in the second column) 

The ground truth can be obtained from publicly available malware datasets. The one in *`data/malheurReference_gt.tsv`* comes from the [Malheur](http://www.mlsec.org/malheur/) dataset. There are other public datasets with ground truth such as [Drebin](https://www.sec.cs.tu-bs.de/~danarp/drebin/) and [Malicia](http://malicia-project.com/dataset.html).

### Preparation

#### Generic Token Detection

The labeling takes as input a file with generic tokens that should be ignored in the AV labels, e.g., trojan, virus, generic, linux. By default, the labeling uses the *`data/default.generics`* generic tokens file. You can edit that file to add additional generic tokens you feel we are missing.

In the original AVCLASS paper [1] presents an automatic approach to identify generic tokens, which **requires ground truth**, i.e., it requires knowing the true family for each input sample.
Not only that, but **the ground truth should be large**, i.e., contain at least one hundred thousand samples. In the evaluation, AVCLASS identified generic tokens using as ground truth the concatenation of all datasets for which we had ground truth. This requirement of a large ground truth dataset is why we expect most users will skip this step and simply use our provided default file.

If you want to test generic token detection you can do:

```
python avclass_generic_detect.py -lb data/malheurReference_lb.json -gt data/malheurReference_gt.tsv -tgen 10 > malheurReference.gen 
```

Each line in the *data/malheurReference_gt.tsv* file has two **tab-separated** columns:

```
0058780b175c3ce5e244f595951f611b8a24bee2 CASINO
```

which indicates that sample 0058780b175c3ce5e244f595951f611b8a24bee2 is known to be of the *CASINO* family.

The *`-tgen 10`* switch is a threshold for the minimum number of families where a token has to be observed to be considered generic. If the switch is ommitted, the default threshold of 8 is used.

The above command outputs two files: *`malheurReference.gen`* and *`malheurReference_lb.gen`*. Each of them has 2 columns: token and number of families where the token was observed. File *`malheurReference.gen`* is the final output with the detected generic tokens for which the number of families is above the given threshold. The file *`malheurReference_lb.gen`* has this information for all tokens. Thus, *`malheurReference.gen`* is a subset of *`malheurReference_lb.gen`*. 

However, note that in the above command you are trying to identify generic tokens from a small dataset since Drebin only contains 3K labeled samples. Thus, *`malheurReference.gen`* only contains 25 identified generic tokens. Using those 25 generic tokens will produce significantly worse results than using the generic tokens in *`data/default.generics`*.

#### Alias Detection

Different vendors may assign different names (i.e., aliases) for the same family. For example, some vendors may use *zeus* and others *zbot* as aliases for the same malware family. The labeling takes as input a file with aliases that should be merged. By default, the labeling uses the *data/default.aliases* aliases file. You can edit that file to add additional aliases you feel we are missing.

In the original AVCLASS paper [1] describes an automatic approach to identify aliases.
Note that the alias detection approach **requires as input the AV labels for large set of samples**, 
e.g., several million samples. In contrast with the generic token detection, the input samples for alias detection **do not need to be labeled**, i.e., no need to know their family. In the evaluation, AVCLASS identified aliases using as input the largest of unlabeled datasets, which contained nearly 8M samples. This requirement of a large input dataset is why we expect most users will skip this step and simply use our provided default file.

If you want to test alias detection you can do:

```
python avclass_alias_detect.py -lb data/malheurReference_lb.json -nalias 100 -talias 0.98 > malheurReference.aliases
```

The `-nalias` threshold provides the minimum number of samples two tokens need to be observed in to be considered aliases. If the switch is not provided the default is 20.

The `-talias` threshold provides the minimum fraction of times that the samples appear together. If the switch is not provided the default is 0.94 (94%).

The above command outputs two files: *`malheurReference.aliases`* and *`malheurReference_lb.alias`*. Each of them has 6 columns: 
1. t1: token that is an alias
2. t2: family for which t1 is an alias
3. |t1|: number of input samples where t1 was observed
4. |t2|: number of input samples where t2 was observed
5. |t1^t2|: number of input samples where both t1 and t2 were observed
6. |t1^t2|/|t1|: ratio of input samples where both t1 and t2 

These were observed over the number of input samples where t1 was observed.

File *`malheurReference.aliases`* is the final output with the detected aliases that satisfy the -nalias and -talias thresholds. The file *`malheurReference_lb.alias`* has this information for all tokens. Thus, *`malheurReference.aliases`* is a subset of *`malheurReference_lb.alias`*.

However, note that in the above command you are trying to identify aliases from a small dataset since Drebin only contains 3K samples. Thus, *`malheurReference.aliases`* only contains 6 identified aliases. Using those 6 aliases will produce significantly worse results than using the aliases in *`data/default.aliases`*. As mentioned, to improve the identified aliases you should provide as input several million samples. 

## Acknowledgment

We deeply respect original authors of AVCLASS. Reference with love:

- [1] Marcos Sebastián, Richard Rivera, Platon Kotzias, and Juan Caballero. 2016. AVCLASS: A tool for Massive Malware Labeling. In *Proceedings of the 19th International Symposium on Research in Attacks, Intrusions and Defenses (RAID'16).* 230--253. **(If you wish to cite the original AVCLASS, please cite this paper; if you wish to cite AVCLASS++, just refer to this repository URL)**
- [2] Platon Kotzias, Srdjan Matic, Richard Rivera, and Juan Caballero. 2015. Certified PUP: Abuse in Authenticode Code Signing. In *Proceedings of the 22nd ACM Conference on Computer and Communication Security (CCS'15).* 465--478.

The techniques introduced in AVCLASS++ are based on the following papers:

- [3] Xiaojin Zhu and Zoubin Ghahramani. 2002. Learning from Labeled and Unlabeled Data with Label Propagation. *Technical Report CMU-CALD-02-107, Carnegie Mellon University.*
- [4] Hyrum S. Anderson and Phil Roth. 2018. EMBER: An Open Dataset for Training Static PE Malware Machine Learning Models. *CoRR, abs/1804.04637.*
- [5] Stefano Schiavoni, Federico Maggi, Lorenzo Cavallaro, and Stefano Zanero. 2014. Phoenix: DGA-Based Botnet Tracking and Intelligence. In *Proceeding of the 11th Conference on Detection of Intrusions and Malware & Vulnerability Assessment (DIMVA'14).* 192--211.

## License

The original AVCLASS:

```
MIT License

Copyright (c) 2016 MaliciaLab @ IMDEA Software Institute

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

AVCLASS++ part:

```
SOFTWARE LICENSE AGREEMENT FOR EVALUATION

This SOFTWARE EVALUATION LICENSE AGREEMENT (this "Agreement") is a legal contract between a person who uses or otherwise accesses or installs the Software (“User(s)”), and Nippon Telegraph and Telephone corporation ("NTT").
READ THE TERMS AND CONDITIONS OF THIS AGREEMENT CAREFULLY BEFORE INSTALLING OR OTHERWISE ACCESSING OR USING NTT'S PROPRIETARY SOFTWARE ACCOMPANIED BY THIS AGREEMENT (the "SOFTWARE"). THE SOFTWARE IS COPYRIGHTED AND IT IS LICENSED TO USER UNDER THIS AGREEMENT, NOT SOLD TO USER. BY INSTALLING OR OTHERWISE ACCESSING OR USING THE SOFTWARE, USER ACKNOWLEDGES THAT USER HAS READ THIS AGREEMENT, THAT USER UNDERSTANDS IT, AND THAT USER ACCEPTS AND AGREES TO BE BOUND BY ITS TERMS. IF AT ANY TIME USER IS NOT WILLING TO BE BOUND BY THE TERMS OF THIS AGREEMENT, USER SHOULD  TERMINATE THE INSTALLATION PROCESS, IMMEDIATELY CEASE AND REFRAIN FROM ACCESSING OR USING THE SOFTWARE AND DELETE ANY COPIES USER MAY HAVE. THIS AGREEMENT REPRESENTS THE ENTIRE AGREEMENT BETWEEN USER AND NTT CONCERNING THE SOFTWARE.

 
BACKGROUND
A.	NTT is the owner of all rights, including all patent rights, and copyrights in and to the Software and related documentation listed in Exhibit A to this Agreement.
B.	User wishes to obtain a royalty free license to use the Software to enable User to evaluate, and NTT wishes to grant such a license to User, pursuant and subject to the terms and conditions of this Agreement.
C.	As a condition to NTT's provision of the Software to User, NTT has required User to execute this Agreement.
In consideration of these premises, and the mutual promises and conditions in this Agreement, the parties hereby agree as follows:
1.	Grant of Evaluation License.  	NTT hereby grants to User, and User hereby accepts, under the terms and conditions of this Agreement, a royalty free, nontransferable and nonexclusive license to use the Software internally for the purposes of testing, analyzing, and evaluating the methods or mechanisms as shown in "Yuma Kurogome, AVCLASS++: Yet Another Massive Malware Labeling Tool, Black Hat Europe Arsenal 2019". User may make a reasonable number of backup copies of the Software solely for User's internal use pursuant to the license granted in this Section 1.
2.  Shipment and Installation.  NTT will ship or deliver the Software by any method that NTT deems appropriate. User shall be solely responsible for proper installation of the Software.
3.  Term.  This Agreement is effective whichever is earlier (i) upon User’s acceptance of the Agreement, or (ii) upon User’s installing, accessing, and using the Software, even if User has not expressly accepted this Agreement. Without prejudice to any other rights, NTT may terminate this Agreement without notice to User. User may terminate this Agreement at any time by User’s decision to terminate the Agreement to NTT and ceasing use of the Software. Upon any termination or expiration of this Agreement for any reason, User agrees to uninstall the Software and destroy all copies of the Software.
4.	Proprietary Rights
(a)	The Software is the valuable and proprietary property of NTT, and NTT shall retain exclusive title to this property both during the term and after the termination of this Agreement.  Without limitation, User acknowledges that all patent rights and copyrights in the Software shall remain the exclusive property of NTT at all times. User shall use not less than reasonable care in safeguarding the confidentiality of the Software. 
(b)	USER SHALL NOT, IN WHOLE OR IN PART, AT ANY TIME DURING THE TERM OF OR AFTER THE TERMINATION OF THIS AGREEMENT: (i) SELL, ASSIGN, LEASE, DISTRIBUTE, OR OTHERWISE TRANSFER THE SOFTWARE TO ANY THIRD PARTY; (ii) EXCEPT AS OTHERWISE PROVIDED HEREIN, COPY OR REPRODUCE THE SOFTWARE IN ANY MANNER; OR (iii) ALLOW ANY PERSON OR ENTITY TO COMMIT ANY OF THE ACTIONS DESCRIBED IN (i) THROUGH (ii) ABOVE.
(c)	User shall take appropriate action, by instruction, agreement, or otherwise, with respect to its employees permitted under this Agreement to have access to the Software to ensure that all of User's obligations under this Section 4 shall be satisfied.  
5.  Indemnity.  User shall defend, indemnify and hold harmless NTT, its agents and employees, from any loss, damage, or liability arising in connection with User's improper or unauthorized use of the Software. NTT SHALL HAVE THE SOLE RIGHT TO CONDUCT DEFEND ANY ACTTION RELATING TO THE SOFTWARE.
6.	Disclaimer.  THE SOFTWARE IS LICENSED TO USER "AS IS," WITHOUT ANY TRAINING, MAINTENANCE, OR SERVICE OBLIGATIONS WHATSOEVER ON THE PART OF NTT. NTT MAKES NO EXPRESS OR IMPLIED WARRANTIES OF ANY TYPE WHATSOEVER, INCLUDING WITHOUT LIMITATION THE IMPLIED WARRANTIES OF MERCHANTABILITY, OF FITNESS FOR A PARTICULAR PURPOSE AND OF NON-INFRINGEMENT ON COPYRIGHT OR ANY OTHER RIGHT OF THIRD PARTIES.  USER ASSUMES ALL RISKS ASSOCIATED WITH ITS USE OF THE SOFTWARE, INCLUDING WITHOUT LIMITATION RISKS RELATING TO QUALITY, PERFORMANCE, DATA LOSS, AND UTILITY IN A PRODUCTION ENVIRONMENT. 
7.	Limitation of Liability.  IN NO EVENT SHALL NTT BE LIABLE TO USER OR TO ANY THIRD PARTY FOR ANY INDIRECT, SPECIAL, INCIDENTAL, OR CONSEQUENTIAL DAMAGES, INCLUDING BUT NOT LIMITED TO DAMAGES FOR PERSONAL INJURY, PROPERTY DAMAGE, LOST PROFITS, OR OTHER ECONOMIC LOSS, ARISING IN CONNECTION WITH USER'S USE OF OR INABILITY TO USE THE SOFTWARE, IN CONNECTION WITH NTT'S PROVISION OF OR FAILURE TO PROVIDE SERVICES PERTAINING TO THE SOFTWARE, OR AS A RESULT OF ANY DEFECT IN THE SOFTWARE.  THIS DISCLAIMER OF LIABILITY SHALL APPLY REGARD¬LESS OF THE FORM OF ACTION THAT MAY BE BROUGHT AGAINST NTT, WHETHER IN CONTRACT OR TORT, INCLUDING WITHOUT LIMITATION ANY ACTION FOR NEGLIGENCE.  USER'S SOLE REMEDY IN THE EVENT OF ANY BREACH OF THIS AGREEMENT BY NTT SHALL BE TERMINATION PURSUANT TO SECTION 3.
8.	No Assignment or Sublicense.  Neither this Agreement nor any right or license under this Agreement, nor the Software, may be sublicensed, assigned, or otherwise transferred by User without NTT's prior written consent.
9.	General
(a)	If any provision, or part of a provision, of this Agreement is or becomes illegal, unenforceable, or invalidated, by operation of law or otherwise, that provision or part shall to that extent be deemed omitted, and the remainder of this Agreement shall remain in full force and effect.
(b)	This Agreement is the complete and exclusive statement of the agreement between the parties with respect to the subject matter hereof, and supersedes all written and oral contracts, proposals, and other communications between the parties relating to that subject matter.  
(c)	Subject to Section 8, this Agreement shall be binding on, and shall inure to the benefit of, the respective successors and assigns of NTT and User.  
(d)	If either party to this Agreement initiates a legal action or proceeding to enforce or interpret any part of this Agreement, the prevailing party in such action shall be entitled to recover, as an element of the costs of such action and not as damages, its attorneys' fees and other costs associated with such action or proceeding.
(e)	This Agreement shall be governed by and interpreted under the laws of Japan, without reference to conflicts of law principles. All disputes arising out of or in connection with this Agreement shall be finally settled by arbitration in Tokyo in accordance with the Commercial Arbitration Rules of the Japan Commercial Arbitration Association.  The arbitration shall be conducted by three (3) arbitrators and in Japanese. The award rendered by the arbitrators shall be final and binding upon the parties. Judgment upon the award may be entered in any court having jurisdiction thereof.
(f)	NTT shall not be liable to the User or to any third party for any delay or failure to perform NTT’s obligation set forth under this Agreement due to any cause beyond NTT’s reasonable control.
 
EXHIBIT A
```
