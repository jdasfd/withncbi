# Process plastid genomes

[TOC levels=1-3]: # ""

- [Process plastid genomes](#process-plastid-genomes)
- [Work flow.](#work-flow)
- [Update taxdmp](#update-taxdmp)
- [Scrap id and acc from NCBI](#scrap-id-and-acc-from-ncbi)
- [Add lineage information](#add-lineage-information)
- [Filtering based on valid families and genera](#filtering-based-on-valid-families-and-genera)
- [Find a way to name these](#find-a-way-to-name-these)
- [Download sequences and regenerate lineage information](#download-sequences-and-regenerate-lineage-information)
  - [Numbers for higher ranks](#numbers-for-higher-ranks)
  - [Raw phylogenetic tree by MinHash](#raw-phylogenetic-tree-by-minhash)
- [Prepare sequences for lastz](#prepare-sequences-for-lastz)
- [Aligning without outgroups](#aligning-without-outgroups)
  - [Create `plastid_t_o.md` for picking targets and outgroups.](#create-plastid_t_omd-for-picking-targets-and-outgroups)
  - [Create alignments plans without outgroups](#create-alignments-plans-without-outgroups)
  - [Plans for align-able targets](#plans-for-align-able-targets)
  - [Plans for diverged families in Angiosperms and Gymnosperms](#plans-for-diverged-families-in-angiosperms-and-gymnosperms)
  - [Aligning w/o outgroups](#aligning-wo-outgroups)
  - [Aligning with outgroups](#aligning-with-outgroups)
  - [Self alignments](#self-alignments)
- [LSC and SSC](#lsc-and-ssc)
  - [Can't get clear IR information](#cant-get-clear-ir-information)
  - [Slices of IR, LSC and SSC](#slices-of-ir-lsc-and-ssc)
- [Summary](#summary)
  - [Copy xlsx files](#copy-xlsx-files)
  - [Genome list](#genome-list)
  - [Statistics of genome alignments](#statistics-of-genome-alignments)
  - [Groups](#groups)
  - [Phylogenic trees of each genus with outgroup](#phylogenic-trees-of-each-genus-with-outgroup)
  - [d1, d2](#d1-d2)


The following command lines are about how I processed the plastid genomes of green plants. Many
tools of `taxon/` are used here, which makes a good example for users.

# Work flow.

```text
id ---> lineage ---> filtering ---> naming ---> strain_info.pl ---> mash ---> egaz
                                      |                              ^
                                      |-------> downloads      ------|
```

# Update taxdmp

*Update `~/data/NCBI/taxdmp` before running `strain_info.pl`*.

# Scrap id and acc from NCBI

Open browser and visit
[NCBI plastid page](http://www.ncbi.nlm.nih.gov/genomes/GenomesGroup.cgi?taxid=33090&opt=plastid).
Save page to a local file, html only. In this case, it's `doc/green_plants_plastid_200326.html`.

All [Eukaryota](http://www.ncbi.nlm.nih.gov/genomes/GenomesGroup.cgi?opt=plastid&taxid=2759),
`doc/eukaryota_plastid_200326.html`.

Or [this link](http://www.ncbi.nlm.nih.gov/genome/browse/?report=5).

```text
Eukaryota (2759)                4711
    Viridiplantae (33090)       4437
        Chlorophyta (3041)      151
        Streptophyta (35493)    4284
```

Use `taxon/id_seq_dom_select.pl` to extract Taxonomy ids and genbank accessions from all history
pages.

```csv
id,acc
996148,NC_017006
```

Got **4755** accessions.

```bash
mkdir -p ~/data/plastid/GENOMES
cd ~/data/plastid/GENOMES

rm webpage_id_seq.csv

perl ~/Scripts/withncbi/taxon/id_seq_dom_select.pl \
    ~/Scripts/withncbi/doc/eukaryota_plastid_200326.html \
    >> webpage_id_seq.csv

perl ~/Scripts/withncbi/taxon/id_seq_dom_select.pl \
    ~/Scripts/withncbi/doc/green_plants_plastid_200326.html \
    >> webpage_id_seq.csv

```

Use `taxon/gb_taxon_locus.pl` to extract information from refseq genbank files.

```bash
cd ~/data/plastid/GENOMES

wget -N ftp://ftp.ncbi.nlm.nih.gov/genomes/refseq/plastid/plastid.1.genomic.gbff.gz
wget -N ftp://ftp.ncbi.nlm.nih.gov/genomes/refseq/plastid/plastid.2.genomic.gbff.gz
wget -N ftp://ftp.ncbi.nlm.nih.gov/genomes/refseq/plastid/plastid.3.genomic.gbff.gz

gzip -dcf plastid.*.genomic.gbff.gz > genomic.gbff

perl ~/Scripts/withncbi/taxon/gb_taxon_locus.pl genomic.gbff > refseq_id_seq.csv

rm genomic.gbff
#rm *.genomic.gbff.gz

cat refseq_id_seq.csv | grep -v "^#" | wc -l
# 4722

# combine
cat webpage_id_seq.csv refseq_id_seq.csv |
    sort -u | # duplicated id-seq pair
    sort -t, -k1,1 \
    > id_seq.csv

cat id_seq.csv | grep -v "^#" | wc -l
# 4755

```

# Add lineage information

Give ids better shapes for manually checking and automatic filtering.

If you sure, you can add or delete lines and contents in `CHECKME.tsv`.

```shell
mkdir -p ~/data/mito/summary
cd ~/data/mito/summary

# generate a TSV file for manually checking
cat ../GENOMES/plant_id_seq.tsv |
    nwr append stdin |
    nwr append stdin -r species -r genus -r family -r order -r class -r phylum |
    keep-header -- sort -k9,9 -k8,8 -k7,7 -k6,6 -k5,5 \
    > CHECKME.tsv

```

Manually correct lineages.

Taxonomy information from [AlgaeBase](http://www.algaebase.org),
[Wikipedia](https://www.wikipedia.org/) and [Encyclopedia of Life](http://eol.org/).


Split Streptophyta according to classical plant classification.

* Streptophyta
  * Streptophytina
    * Embryophyta
      * Tracheophyta
        * Euphyllophyta
          * Spermatophyta
            * Magnoliopsida (flowering plants) 3398 - Angiosperm
            * Acrogymnospermae 1437180 - Gymnosperm
          * Polypodiopsida 241806 - ferns
        * Lycopodiopsida 1521260 - clubmosses
      * Anthocerotophyta 13809 - hornworts
      * Bryophyta 3208 - mosses
      * Marchantiophyta 3195 - liverworts

```bash
cd ~/data/plastid/summary

# darwin (bsd) need "" for -i
sed -i".bak" CHECKME.csv

# Entry Merged. Taxid 1605147 was merged into taxid 142389 on October 16, 2015.
sed -i".bak" "/1605147,/d" CHECKME.csv

# comma or parenthesis in names
sed -i".bak" "/167339,/d" CHECKME.csv

# missing all
sed -i".bak" "/2003521,/d" CHECKME.csv

# Angiosperms
nwr member 3398 -r family |
    tsv-select -f 2 |
    parallel -r -j 1 '
        perl -pi -e '\''
            s/({}\t\w+\t\w+)\tStreptophyta/\1\tAngiosperms/g
        '\'' CHECKME.tsv
    '

# Gymnosperms
nwr member 1437180 -r family |
    tsv-select -f 2 |
    parallel -r -j 1 '
        perl -pi -e '\''
            s/({}\t\w+\t\w+)\tStreptophyta/\1\tGymnosperms/g
        '\'' CHECKME.tsv
    '

# Pteridophytes
nwr member 241806 -r family |
    tsv-select -f 2 |
    parallel -r -j 1 '
        perl -pi -e '\''
            s/({}\t\w+\t\w+)\tStreptophyta/\1\tPteridophytes/g
        '\'' CHECKME.tsv
    '
nwr member 1521260 -r family |
    tsv-select -f 2 |
    parallel -r -j 1 '
        perl -pi -e '\''
            s/({}\t\w+\t\w+)\tStreptophyta/\1\tPteridophytes/g
        '\'' CHECKME.tsv
    '

# Bryophytes
nwr member 13809 -r family |
    tsv-select -f 2 |
    parallel -r -j 1 '
        perl -pi -e '\''
            s/({}\t\w+\t\w+)\tStreptophyta/\1\tBryophytes/g
        '\'' CHECKME.tsv
    '

nwr member 3208 -r family |
    tsv-select -f 2 |
    parallel -r -j 1 '
        perl -pi -e '\''
            s/({}\t\w+\t\w+)\tStreptophyta/\1\tBryophytes/g
        '\'' CHECKME.tsv
    '

nwr member 3195 -r family |
    tsv-select -f 2 |
    parallel -r -j 1 '
        perl -pi -e '\''
            s/({}\t\w+\t\w+)\tStreptophyta/\1\tBryophytes/g
        '\'' CHECKME.tsv
    '

# Charophyta 轮藻
parallel -j 1 '
    sed -i".bak" "s/{}\tStreptophyta/{}\tCharophyta/" CHECKME.tsv
    ' ::: \
        Charophyceae Chlorokybophyceae Coleochaetophyceae \
        Zygnemophyceae

# Chlorophyta 绿藻
parallel -j 1 '
    sed -i".bak" "s/{}\tStreptophyta/{}\tChlorophyta/" CHECKME.tsv
    ' ::: \
        Mesostigmatophyceae Klebsormidiophyceae

# Ochrophyta 褐藻
parallel -j 1 '
    sed -i".bak" "s/{}\tNA/{}\tOchrophyta/" CHECKME.tsv
    ' ::: \
        Bolidophyceae Dictyochophyceae Eustigmatophyceae \
        Pelagophyceae Phaeophyceae Raphidophyceae \
        Synurophyceae Xanthophyceae

# Rhodophyta 红藻
parallel -j 1 '
    sed -i".bak" "s/{}\tNA/{}\tRhodophyta/" CHECKME.tsv
    ' ::: \
        Bangiophyceae Compsopogonophyceae Florideophyceae \
        Rhodellophyceae Stylonematophyceae

# Cryptophyta 隐藻
parallel -j 1 '
    sed -i".bak" "s/{}\tNA/{}\tCryptophyta/" CHECKME.tsv
    ' ::: \
        Cryptophyta Cryptophyceae

# missing phylums
sed -i".bak" "s/Glaucocystophyceae\tNA/Glaucocystophyceae\tGlaucophyta/" CHECKME.tsv
sed -i".bak" "s/Dinophyceae\tNA/Dinophyceae\tDinoflagellata/" CHECKME.tsv

# Chrysanthemum x morifolium and Pelargonium x hortorum are also weird, but they can be googled.

rm *.tmp *.bak

```

# Filtering based on valid families and genera

Species and genus should not be "NA" and genus has 2 or more members.

```text
4754 ---------> 4725 ---------> 3338 ---------> 4290
        NA             genus          family
```

```bash
mkdir -p ~/data/plastid/summary
cd ~/data/plastid/summary

cat CHECKME.csv | grep -v "^#" | wc -l
# 4754

# filter out accessions without linage information (strain, species, genus and family)
cat CHECKME.csv |
    perl -nla -F"," -e '
        /^#/ and next;
        ($F[2] eq q{NA} or $F[3] eq q{NA} or $F[4] eq q{NA} or $F[5] eq q{NA} ) and next;
        print
    ' \
    > valid.tmp

wc -l valid.tmp
# 4725

#----------------------------#
# Genus
#----------------------------#
# valid genera
cat valid.tmp |
    perl -nla -F"," -e '
        $seen{$F[4]}++;
        END {
            for $k (sort keys %seen) {
                printf qq{,%s,\n}, $k if $seen{$k} > 1
            }
        }
    ' \
    > genus.tmp

# intersect between two files
grep -F -f genus.tmp valid.tmp > valid.genus.tmp

wc -l valid.genus.tmp
# 3338

#----------------------------#
# Family
#----------------------------#
# get some genera back as candidates for outgroup
cat valid.genus.tmp |
    perl -nla -F"," -e 'printf qq{,$F[5],\n}' \
    > family.tmp

# intersect between two files
grep -F -f family.tmp valid.tmp > valid.family.tmp

wc -l valid.family.tmp
# 4290

#----------------------------#
# results produced in this step
#----------------------------#
head -n 1 CHECKME.csv > DOWNLOAD.csv
cat valid.family.tmp >> DOWNLOAD.csv

# clean
rm *.tmp *.bak

```

# Find a way to name these

Seems it's OK to use species as names.

```bash
cd ~/data/plastid/summary

# sub-species
cat DOWNLOAD.csv |
    perl -nl -a -F"," -e '
        /^#/i and next;
        $seen{$F[3]}++;
        END {
            for $k (keys %seen){printf qq{%s,%d\n}, $k, $seen{$k} if $seen{$k} > 1}
        };
    ' |
    sort
#Allium cyathophorum,2
#Anemone hepatica,2
#Arabidopsis lyrata,2
#Astragalus mongholicus,2
#Brassica rapa,2
#Cannabis sativa,2
#Capsicum baccatum,3
#Changiostyrax dolichocarpus,2
#Corallorhiza striata,2
#Corylus ferox,2
#Fragaria vesca,2
#Gossypium herbaceum,2
#Hippophae rhamnoides,2
#Hordeum vulgare,2
#Magnolia officinalis,2
#Marchantia polymorpha,2
#Musa balbisiana,2
#Olea europaea,4
#Oryza sativa,4
#Panax japonicus,2
#Paris polyphylla,2
#Physcomitrella patens,2
#Pisum sativum,2
#Plasmodium falciparum,2
#Saccharum hybrid cultivar,3
#Sanguisorba tenuifolia,2
#Sinalliaria limprichtiana,2
#Solanum bukasovii,2
#Solanum lycopersicum,2
#Solanum stenotomum,2
#Sophora alopecuroides,2
#Vitis aestivalis,2
#Vitis cinerea,4
#Vitis rotundifolia,2
#Wurfbainia villosa,2

# strain name not equal to species
cat DOWNLOAD.csv |
    grep -v '^#' |
    perl -nl -a -F"," -e '$F[2] ne $F[3] and print $F[2]' |
    sort
#Actinidia callosa var. henryi
#Allium cyathophorum var. farreri
#Anemone cernua var. koreana
#Anemone hepatica var. asiatica
#Anemone hepatica var. japonica
#Arabidopsis lyrata subsp. lyrata
#Astragalus mongholicus var. nakaianus
#Babesia bovis T2Bo
#Babesia microti strain RI
#Betula pendula var. carelica
#Brassica rapa subsp. pekinensis
#Calycanthus floridus var. glaucus
#Calypso bulbosa var. occidentalis
#Capsicum baccatum var. baccatum
#Capsicum baccatum var. pendulum
#Capsicum baccatum var. praetermissum
#Caragana rosea var. rosea
#Corallorhiza striata var. involuta
#Corallorhiza striata var. striata
#Corylus ferox var. thibetica
#Cucumis melo subsp. melo
#Eucalyptus globulus subsp. globulus
#Fagopyrum esculentum subsp. ancestrale
#Fragaria vesca subsp. bracteata
#Fragaria vesca subsp. vesca
#Gossypium herbaceum subsp. africanum
#Gracilaria tenuistipitata var. liui
#Hippophae rhamnoides subsp. yunnanensis
#Hordeum vulgare subsp. spontaneum
#Hordeum vulgare subsp. vulgare
#Lilium martagon var. pilosiusculum
#Magnolia macrophylla var. dealbata
#Magnolia officinalis subsp. biloba
#Marchantia polymorpha subsp. ruderalis
#Musa balbisiana var. balbisiana
#Oenothera elata subsp. hookeri
#Olea europaea subsp. cuspidata
#Olea europaea subsp. europaea
#Olea europaea subsp. maroccana
#Olea woodiana subsp. woodiana
#Oryza sativa f. spontanea
#Oryza sativa Indica Group
#Oryza sativa Japonica Group
#Panax japonicus var. bipinnatifidus
#Paris polyphylla var. chinensis
#Paris polyphylla var. yunnanensis
#Phalaenopsis aphrodite subsp. formosana
#Phryma leptostachya subsp. asiatica
#Phyllostachys nigra var. henonis
#Pisum sativum subsp. elatius
#Plasmodium chabaudi chabaudi
#Plasmodium falciparum 3D7
#Plasmodium falciparum HB3
#Pseudotsuga sinensis var. wilsoniana
#Rosa chinensis var. spontanea
#Saccharum hybrid cultivar NCo 310
#Saccharum hybrid cultivar SP80-3280
#Sanguisorba tenuifolia var. alba
#Sinalliaria limprichtiana var. grandifolia
#Solanum bukasovii f. multidissectum
#Solanum stenotomum subsp. goniocalyx
#Sophora alopecuroides var. alopecuroides
#Styphnolobium japonicum var. japonicum
#Thalassiosira oceanica CCMP1005
#Trichopus zeylanicus subsp. travancoricus
#Vitis aestivalis var. linsecomii
#Vitis cinerea var. cinerea
#Vitis cinerea var. floridana
#Vitis cinerea var. helleri
#Vitis rotundifolia var. munsoniana
#Wurfbainia villosa var. xanthioides

```

Create abbreviations.

```bash
cd ~/data/plastid/summary

echo '#strain_taxon_id,accession,strain,species,genus,family,order,class,phylum,abbr' > ABBR.csv
cat DOWNLOAD.csv |
    grep -v '^#' |
    perl ~/Scripts/withncbi/taxon/abbr_name.pl -c "3,4,5" -s "," -m 0 --shortsub |
    sort -t',' -k9,9 -k7,7 -k6,6 -k10,10 \
    >> ABBR.csv

```

# Download sequences and regenerate lineage information

We don't rename sequences here, so the file has three columns.

And create `taxon_ncbi.csv` with abbr names as taxon file.

```bash
cd ~/data/plastid/GENOMES

echo "#strain_name,accession,strain_taxon_id" > name_acc_id.csv
cat ../summary/ABBR.csv |
    grep -v '^#' |
    perl -nl -a -F"," -e 'print qq{$F[9],$F[1],$F[0]}' |
    sort \
    >> name_acc_id.csv

# local, Runtime 10 seconds.
# with --entrez, Runtime 7 minutes and 23 seconds.
# And which-can't-find is still which-can't-find.
cat ../summary/ABBR.csv |
    grep -v '^#' |
    perl -nla -F"," -e 'print qq{$F[0],$F[9]}' |
    uniq |
    perl ~/Scripts/withncbi/taxon/strain_info.pl --stdin --withname --file taxon_ncbi.csv

# Some warnings about trans-splicing genes from BioPerl, just ignore them
# eutils restricts 3 connections
cat name_acc_id.csv |
    grep -v '^#' |
    2>&1 parallel --colsep ',' --no-run-if-empty --linebuffer -k -j 3 "
        echo -e '==> id: [{1}]\tseq: [{2}]\n'
        mkdir -p {1}
        if [[ -e '{1}/{2}.gff' && -e '{1}/{2}.fa' ]] ; then
            echo -e '    Sequence [{1}/{2}] exists, next\n'
            exit
        fi

        # gb
        echo -e '    [{1}/{2}].gb'
        curl -Ls \
            'http://eutils.ncbi.nlm.nih.gov/entrez/eutils/efetch.fcgi?db=nucleotide&id={2}&rettype=gb&retmode=text' \
            > {1}/{2}.gb

        # fasta
        echo -e '    [{1}/{2}].fa'
        curl -Ls \
            'http://eutils.ncbi.nlm.nih.gov/entrez/eutils/efetch.fcgi?db=nucleotide&id={2}&rettype=fasta&retmode=text' \
            > {1}/{2}.fa

        # gff
        echo -e '    [{1}/{2}].gff'
        perl ~/Scripts/withncbi/taxon/bp_genbank2gff3.pl {1}/{2}.gb -o stdout > {1}/{2}.gff
        perl -i -nlp -e '/^\#\#FASTA/ and last' {1}/{2}.gff

        echo
    " |
    tee download_seq.log

# count downloaded sequences
find . -name "*.fa" | wc -l

```

## Numbers for higher ranks

92 orders, 202 families, 636 genera and 3295 species.

```bash
cd ~/data/plastid/summary

# valid genera
cat ABBR.csv |
    grep -v "^#" |
    perl -nl -a -F"," -e '
        $seen{$F[4]}++;
        END {
            for $k (sort keys %seen) {
                printf qq{,%s,\n}, $k if $seen{$k} > 1
            }
        }
    ' \
    > genus.valid.tmp

# intersect between two files
grep -F -f genus.valid.tmp ABBR.csv > GENUS.csv

wc -l GENUS.csv
# 113

# count every ranks
cut -d',' -f 4 GENUS.csv | sort | uniq > species.list.tmp
cut -d',' -f 5 GENUS.csv | sort | uniq > genus.list.tmp
cut -d',' -f 6 GENUS.csv | sort | uniq > family.list.tmp
cut -d',' -f 7 GENUS.csv | sort | uniq > order.list.tmp
wc -l order.list.tmp family.list.tmp genus.list.tmp species.list.tmp
#   92 order.list.tmp
#  202 family.list.tmp
#  636 genus.list.tmp
# 3295 species.list.tmp

# sort by multiply columns, phylum, order, family, genus, accession
# oldest accession will be the target
head -n 1 ABBR.csv > GENUS.tmp
cat GENUS.csv |
    sort -t',' -k9,9 -k7,7 -k6,6 -k5,5 -k2,2 \
    >> GENUS.tmp

mv GENUS.tmp GENUS.csv

# clean
rm *.tmp *.bak

```

## Raw phylogenetic tree by MinHash

```bash
mkdir -p ~/data/plastid/mash
cd ~/data/plastid/mash

cat ../summary/ABBR.csv | sed -e '1d' | cut -d"," -f 10 | sort |
    parallel --no-run-if-empty --linebuffer -k -j 6 '
        2>&1 echo "==> {}"

        if [[ -e {}.msh ]]; then
            exit;
        fi

        find ../GENOMES/{} -name "*.fa" |
            xargs cat |
            mash sketch -k 21 -s 100000 -p 4 - -I "{}" -o {}
    '

```

* h=0.1 as we need outgroup ~ 0.05

```bash
cd ~/data/plastid/summary
mash triangle -E -p 8 -l <(
        cat ABBR.csv |
            sed '1d' |
            cut -d"," -f 10 |
            parallel echo "../mash/{}.msh"
    ) \
    > dist.tsv

# fill matrix with lower triangle
tsv-select -f 1-3 dist.tsv |
    (tsv-select -f 2,1,3 dist.tsv && cat) |
    (
        cut -f 1 dist.tsv |
            tsv-uniq |
            parallel -j 1 --keep-order 'echo -e "{}\t{}\t0"' &&
        cat
    ) \
    > dist_full.tsv

cat dist_full.tsv |
    Rscript -e '
        library(readr);
        library(tidyr);
        library(ape);
        pair_dist <- read_tsv(file("stdin"), col_names=F);
        tmp <- pair_dist %>%
            pivot_wider( names_from = X2, values_from = X3, values_fill = list(X3 = 1.0), values_fn = list(X3 = mean) )
        tmp <- as.matrix(tmp)
        mat <- tmp[,-1]
        rownames(mat) <- tmp[,1]

        dist_mat <- as.dist(mat)
        clusters <- hclust(dist_mat, method = "ward.D2")
        tree <- as.phylo(clusters)
        write.tree(phy=tree, file="tree.nwk")

        group <- cutree(clusters, h=0.1)
        groups <- as.data.frame(group)
        groups$ids <- rownames(groups)
        rownames(groups) <- NULL
        groups <- groups[order(groups$group), ]
        write_tsv(groups, "groups.tsv")
    '

nw_display -s -b 'visibility:hidden' -w 600 -v 30 tree.nwk |
    rsvg-convert -o plastid.png

```

* Abnormal strains

```bash
cd ~/data/plastid/summary

# genus
cut -d',' -f 5 GENUS.csv | sed -e '1d' | uniq |
    parallel -j 1 -k '
        group=$(
            tsv-join groups.tsv -d 2 \
                -f <(cat GENUS.csv | grep -w {} | cut -d, -f 10) \
                -k 1 |
                cut -f 1 |
                sort |
                uniq
        )
        number=$(echo "${group}" | wc -l)
        echo -e "{},${number}"
    ' |
    tsv-join --delimiter ","  -d 1 -f GENUS.csv -k 5 -a 9 |
    tr "," "\t" |
    tsv-filter --ne 2:1
#Actaea  2       Angiosperms
#Adenophora      2       Angiosperms
#Asarum  2       Angiosperms
#Burmannia       3       Angiosperms
#Cuscuta 4       Angiosperms
#Cypripedium     2       Angiosperms
#Dendrobium      2       Angiosperms
#Epipogium       2       Angiosperms
#Eragrostis      2       Angiosperms
#Erodium 2       Angiosperms
#Genlisea        2       Angiosperms
#Gordonia        2       Angiosperms
#Gossypium       2       Angiosperms
#Gynochthodes    2       Angiosperms
#Heritiera       2       Angiosperms
#Ipomoea 2       Angiosperms
#Lagerstroemia   2       Angiosperms
#Lobelia 3       Angiosperms
#Monotropa       2       Angiosperms
#Neottia 2       Angiosperms
#Passiflora      2       Angiosperms
#Pedicularis     2       Angiosperms
#Pelargonium     2       Angiosperms
#Pilostyles      2       Angiosperms
#Torricellia     2       Angiosperms
#...
#Pinus   2       Gymnosperms
#...

# family
cut -d',' -f 6 GENUS.csv | sed -e '1d' | uniq |
    parallel -j 1 -k '
        group=$(
            tsv-join groups.tsv -d 2 \
                -f <(cat GENUS.csv | grep -w {} | cut -d, -f 10) \
                -k 1 |
                cut -f 1 |
                sort |
                uniq
        )
        number=$(echo "${group}" | wc -l)
        echo -e "{},${number}"
    ' |
    tsv-join --delimiter ","  -d 1 -f GENUS.csv -k 6 -a 9 |
    tr "," "\t" |
    tsv-filter --ne 2:1
#Araceae 2       Angiosperms
#Torricelliaceae 2       Angiosperms
#Amaryllidaceae  2       Angiosperms
#Asparagaceae    3       Angiosperms
#Orchidaceae     14      Angiosperms
#Asteraceae      5       Angiosperms
#Campanulaceae   7       Angiosperms
#Brassicaceae    2       Angiosperms
#Caryophyllaceae 2       Angiosperms
#Chenopodiaceae  2       Angiosperms
#Apodanthaceae   2       Angiosperms
#Cucurbitaceae   2       Angiosperms
#Burmanniaceae   3       Angiosperms
#Caprifoliaceae  2       Angiosperms
#Ericaceae       3       Angiosperms
#Primulaceae     2       Angiosperms
#Styracaceae     2       Angiosperms
#Theaceae        2       Angiosperms
#Fabaceae        12      Angiosperms
#Rubiaceae       3       Angiosperms
#Geraniaceae     6       Angiosperms
#Lamiaceae       3       Angiosperms
#Lentibulariaceae        2       Angiosperms
#Oleaceae        2       Angiosperms
#Orobanchaceae   7       Angiosperms
#Plantaginaceae  2       Angiosperms
#Liliaceae       2       Angiosperms
#Melanthiaceae   2       Angiosperms
#Passifloraceae  2       Angiosperms
#Malvaceae       3       Angiosperms
#Lythraceae      2       Angiosperms
#Aristolochiaceae        3       Angiosperms
#Poaceae 14      Angiosperms
#Berberidaceae   4       Angiosperms
#Ranunculaceae   6       Angiosperms
#Rosaceae        3       Angiosperms
#Convolvulaceae  6       Angiosperms
#Solanaceae      2       Angiosperms
#...
#Cupressaceae    3       Gymnosperms
#Taxaceae        3       Gymnosperms
#Pinaceae        5       Gymnosperms
#...

```

# Prepare sequences for lastz

```bash
cd ~/data/plastid/GENOMES

find . -maxdepth 1 -mindepth 1 -type d |
    sort |
    parallel --no-run-if-empty --linebuffer -k -j 6 '
        echo >&2 "==> {}"

        if [ -e {}/chr.fasta ]; then
            echo >&2 "    {} has been processed";
            exit;
        fi

        egaz prepseq \
            {} \
            --gi -v --repeatmasker " --gff --parallel 4"
    '

# restore to original states
#for suffix in .2bit .fasta .fasta.fai .sizes .rm.out .rm.gff; do
#    find . -name "*${suffix}" | parallel --no-run-if-empty rm
#done

```

# Aligning without outgroups

## Create `plastid_t_o.md` for picking targets and outgroups.

Manually edit it then move to `~/Scripts/withncbi/doc/plastid_t_o.md`.

* Listed targets were well curated.

* Outgroups can be changes with less intentions.

```bash
cd ~/data/plastid/summary

cat GENUS.csv |
    grep -v "^#" |
    perl -na -F"," -e '
        BEGIN{
            ($phylum, $family, $genus, ) = (q{}, q{}, q{});
        }

        chomp for @F;

        if ($F[8] ne $phylum) {
            $phylum = $F[8];
            printf qq{\n# %s\n}, $phylum;
        }
        if ($F[5] ne $family) {
            $family = $F[5];
            printf qq{## %s\n}, $family;
        }
        $F[4] =~ s/\W+/_/g;
        if ($F[4] ne $genus) {
            $genus = $F[4];
            printf qq{%s,%s\n}, $genus, $F[9];
        }
    ' \
    > plastid_t_o.md

cat plastid_t_o.md |
    grep -v '^#' |
    grep -E '\S+' |
    wc -l
# 636

# mv plastid_t_o.md ~/Scripts/withncbi/doc/plastid_t_o.md

```

## Create alignments plans without outgroups

```text
GENUS.csv
#strain_taxon_id,accession,strain,species,genus,family,order,class,phylum,abbr
```

**636** genera, **174** families, **117** mash groups, and **74** sub-families.

```bash
mkdir -p ~/data/plastid/taxon
cd ~/data/plastid/taxon

echo -e "#Serial\tGroup\tCount\tTarget" > group_target.tsv

# genus
cat ../summary/GENUS.csv |
    grep -v "^#" |
    SERIAL=1 perl -na -F"," -MPath::Tiny -e '
        BEGIN{
            $name = q{};
            %id_of = ();
            %h = ();
            @ls = grep {/\S/}
                  grep {!/^#/}
                  path(q{~/Scripts/withncbi/doc/plastid_t_o.md})->lines({chomp => 1});
            for (@ls) {
                @fs = split(/,/);
                $h{$fs[0]}= $fs[1];
            }
            undef @ls;
        }

        chomp for @F;
        $F[4] =~ s/\W+/_/g;
        if ($F[4] ne $name) {
            if ($name) {
                if (exists $h{$name}) {
                    my @s = sort {$id_of{$a} <=> $id_of{$b}} keys %id_of;
                    my $t = $h{$name};
                    printf qq{%s\t%s\t%s\t%s\n}, $ENV{SERIAL}, $name, scalar @s, $t;
                    path(qq{$name})->spew(map {qq{$_\n}} @s);
                    $ENV{SERIAL}++;
                }
            }
            $name = $F[4];
            %id_of = ();
        }
        $id_of{$F[9]} = $F[0]; # same strain multiple chromosomes collapsed here

        END {
            my @s = sort {$id_of{$a} <=> $id_of{$b}} keys %id_of;
            my $t = $h{$name};
            printf qq{%s\t%s\t%s\t%s\n}, $ENV{SERIAL}, $name, scalar @s, $t;
            path(qq{$name})->spew(map {qq{$_\n}} @s);
        }' \
    >> group_target.tsv

# family
cat ../summary/ABBR.csv |
    grep -v "^#" |
    SERIAL=1001 perl -na -F"," -MPath::Tiny -e '
        BEGIN{
            our $name = q{};
            our %id_of = ();
        }

        chomp for @F;
        my $family = $F[5];
        $family =~ s/\W+/_/g;
        if ($family ne $name) {
            if ($name) {
                # sort by taxonomy_id
                my @s = sort {$id_of{$a} <=> $id_of{$b}} keys %id_of;
                my $t = $s[0];
                if (scalar @s > 2) {
                    printf qq{%s\t%s\t%s\t%s\n}, $ENV{SERIAL}, $name, scalar @s, $t;
                    path(qq{$name})->spew(map {qq{$_\n}} @s);
                    $ENV{SERIAL}++;
                }
            }
            $name = $family;
            %id_of = ();
        }
        $id_of{$F[9]} = $F[0]; # multiple chromosomes collapsed here

        END {
            my @s = sort {$id_of{$a} <=> $id_of{$b}} keys %id_of;
            my $t = $s[0];
            if (scalar @s > 2) {
                printf qq{%s\t%s\t%s\t%s\n}, $ENV{SERIAL}, $name, scalar @s, $t;
                path(qq{$name})->spew(map {qq{$_\n}} @s);
            }
        }
    '  \
    >> group_target.tsv

```

## Plans for align-able targets

```bash
cd ~/data/plastid/taxon

cat ~/Scripts/withncbi/doc/plastid_t_o.md |
    grep -v "^#" |
    grep . |
    perl -nla -F"," -e 'print $F[1] ' \
    > targets.tmp

cat ../summary/groups.tsv |
    grep -F -w -f targets.tmp |
    perl -nla -F"\t" -e '($g, $s) = split q{_}, $F[1]; print qq{$F[0]\t${g}_${s}}' |
    tsv-summarize --group-by 1 --count |
    tsv-filter --ge 2:2 |
    cut -f 1 \
    > groups.tmp

cat ../summary/groups.tsv |
    grep -F -w -f targets.tmp |
    grep -F -w -f groups.tmp |
    tsv-summarize --group-by 1 --values 2 \
    > groups.lst.tmp

cat groups.lst.tmp |
    grep -v "^#" |
    SERIAL=2001 perl -na -F"\t" -MPath::Tiny -e '
        chomp for @F;
        my $group = $F[0];
        $group = "group_${group}";
        my @targets = split /\|/, $F[1];

        printf qq{%s\t%s\t%s\t%s\n}, $ENV{SERIAL}, $group, scalar @targets, $targets[0];
        path(qq{$group})->spew(map {qq{$_\n}} @targets);
        $ENV{SERIAL}++;
    '  \
    >> group_target.tsv

rm *.tmp

```

## Plans for diverged families in Angiosperms and Gymnosperms

```bash
cd ~/data/plastid/taxon

SERIAL=3001
for family in \
    Aristolochiaceae Burmanniaceae Campanulaceae Convolvulaceae \
    Droseraceae Ericaceae Fabaceae Geraniaceae \
    Orchidaceae Orobanchaceae Poaceae \
    Ranunculaceae \
    Cupressaceae Pinaceae Podocarpaceae \
    ; do

    tsv-join ../summary/groups.tsv -d 2 \
        -f ${family} -k 1 |
        tsv-summarize --group-by 1 --count --unique-values 2 |
        tsv-filter --ge 2:3 |
        SERIAL=${SERIAL} FAMILY=${family} perl -na -F"\t" -MPath::Tiny -e '
            chomp for @F;
            my $group = $F[0];
            $group = "$ENV{FAMILY}_${group}";
            my @members = split /\|/, $F[2];

            printf qq{%s\t%s\t%s\t%s\n}, $ENV{SERIAL}, $group, scalar @members, $members[0];
            path(qq{$group})->spew(map {qq{$_\n}} @members);
            $ENV{SERIAL}++;
        '  \
        >> group_target.tsv

    SERIAL=$(tail -n 1 group_target.tsv | cut -f 1)
    ((SERIAL++))
done
unset SERIAL

```

## Aligning w/o outgroups

```bash
cd ~/data/plastid/

# genus
cat taxon/group_target.tsv |
    tsv-filter -H  --ge 1:1 --le 1:1000 |
    sed -e '1d' | #grep -w "^24" |
    parallel --colsep '\t' --no-run-if-empty --linebuffer -k -j 6 '
        echo -e "==> Group: [{2}]\tTarget: [{4}]\n"

        if bjobs -w | grep -w {2}; then
            exit;
        fi

        egaz template \
            GENOMES/{4} \
            $(cat taxon/{2} | grep -v -x "{4}" | xargs -I[] echo "GENOMES/[]") \
            --multi -o groups/genus/{2} \
            --taxon ~/data/organelle/plastid/GENOMES/taxon_ncbi.csv \
            --rawphylo --aligndb --parallel 4 -v

        bash groups/genus/{2}/1_pair.sh
        bash groups/genus/{2}/2_rawphylo.sh
        bash groups/genus/{2}/3_multi.sh
        bash groups/genus/{2}/6_chr_length.sh
        bash groups/genus/{2}/7_multi_aligndb.sh
    '

# family
cat taxon/group_target.tsv |
    tsv-filter -H --ge 1:1001 --le 1:2000 |
    sed -e '1d' | #grep -w "^1008" |
    parallel --colsep '\t' --no-run-if-empty --linebuffer -k -j 3 '
        echo -e "==> Group: [{2}]\tTarget: [{4}]\n"

        egaz template \
            GENOMES/{4} \
            $(cat taxon/{2} | grep -v -x "{4}" | xargs -I[] echo "GENOMES/[]") \
            --multi -o groups/family/{2} \
            --rawphylo --parallel 8 -v

        bash groups/family/{2}/1_pair.sh
        bash groups/family/{2}/2_rawphylo.sh
        bash groups/family/{2}/3_multi.sh
    '

# mash
cat taxon/group_target.tsv |
    tsv-filter -H --ge 1:2001 --le 1:3000 |
    sed -e '1d' | #grep -w "^2001" |
    parallel --colsep '\t' --no-run-if-empty --linebuffer -k -j 6 '
        echo -e "==> Group: [{2}]\tTarget: [{4}]\n"

        egaz template \
            GENOMES/{4} \
            $(cat taxon/{2} | grep -v -x "{4}" | xargs -I[] echo "GENOMES/[]") \
            --multi -o groups/mash/{2} \
            --rawphylo --parallel 4 -v

        bash groups/mash/{2}/1_pair.sh
        bash groups/mash/{2}/2_rawphylo.sh
        bash groups/mash/{2}/3_multi.sh
    '

# diverged families
cat taxon/group_target.tsv |
    tsv-filter -H --ge 1:3001 --le 1:4000 |
    sed -e '1d' | #grep -w "^3001" |
    parallel --colsep '\t' --no-run-if-empty --linebuffer -k -j 3 '
        echo -e "==> Group: [{2}]\tTarget: [{4}]\n"

        egaz template \
            GENOMES/{4} \
            $(cat taxon/{2} | grep -v -x "{4}" | xargs -I[] echo "GENOMES/[]") \
            --multi -o groups/family/{2} \
            --rawphylo --parallel 8 -v

        bash groups/family/{2}/1_pair.sh
        bash groups/family/{2}/2_rawphylo.sh
        bash groups/family/{2}/3_multi.sh
    '

# clean
find groups -mindepth 1 -maxdepth 3 -type d -name "*_raw" | parallel -r rm -fr
find groups -mindepth 1 -maxdepth 3 -type d -name "*_fasta" | parallel -r rm -fr

# check status
echo \
    $(find groups/genus -mindepth 1 -maxdepth 1 -type d | wc -l) \
    $(find groups/genus -mindepth 1 -maxdepth 3 -type f -name "*.nwk.pdf" | grep -w raw -v | wc -l)

find groups/genus -mindepth 1 -maxdepth 1 -type d |
    parallel -j 4 '
        lines=$(find {} -type f -name "*.nwk.pdf" | grep -w raw -v | wc -l)
        if [ $lines -eq 0 ]; then
            lines=$(find {} -type d -name "mafSynNet" | wc -l)
            if [ $lines -gt 2 ]; then
                echo {}
            fi
        fi
    ' |
    sort

```

* Abnormal groups

```bash
cd ~/data/plastid/

find groups/ -name "pairwise.coverage.csv" | sort |
    parallel -j 4 -k '
        cover=$(cat {} | grep -w "intersect" | cut -d, -f 4)

        if [[ "$cover" == "" ]]; then
            cover=$(cat {} | cut -d, -f 4 | tsv-summarize -H --mean 1 | sed -e "1d")
        fi

        taxon=$(
            echo {//} |
                sed -e "s/^groups\///g" -e "s/\/Results//g" |
                sed -e "s/^family\///g" -e "s/^genus\///g" -e "s/^mash\///g"
            )
        phylum=$(
            cat summary/ABBR.csv |
                grep -w ${taxon} |
                cut -d, -f 9 |
                head -n 1
            )
        group=$(
            echo {//} |
                sed -e "s/^groups\///g" -e "s/\/Results//g" |
                sed -E "s/\/.+//g"
            )

        if [ $(bc <<< "${cover} < 0.5") -eq 1 ]; then
            echo -e "${taxon}\t${cover}\t${phylum}\t${group}"
        fi
    ' | tsv-filter --or --str-eq 3:Angiosperms --str-eq 3:Gymnosperms --empty 3

#Burmanniaceae_69        0.4590          family
#Burmanniaceae   0.1446  Angiosperms     family
#Campanulaceae   0.2860  Angiosperms     family
#Convolvulaceae  0.0975  Angiosperms     family
#Cupressaceae_332        0.4688          family
#Cupressaceae    0.4870  Gymnosperms     family
#Droseraceae_52  0.4033          family
#Droseraceae     0.4092  Angiosperms     family
#Ericaceae       0.0287  Angiosperms     family
#Fabaceae        0.2040  Angiosperms     family
#Geraniaceae     0.1921  Angiosperms     family
#Orchidaceae     0.0160  Angiosperms     family
#Orobanchaceae   0.1098  Angiosperms     family
#Pinaceae        0.3881  Gymnosperms     family
#Poaceae 0.4518  Angiosperms     family
#Podocarpaceae   0.3130  Gymnosperms     family
#Ranunculaceae   0.4627  Angiosperms     family
#Asarum  0.0768  Angiosperms     genus
#Burmannia       0.1359  Angiosperms     genus
#Cuscuta 0.2077  Angiosperms     genus
#Drosera 0.3388  Angiosperms     genus
#Erodium 0.4704  Angiosperms     genus
#Lobelia 0.4908  Angiosperms     genus
#Monotropa       0.2901  Angiosperms     genus

```

## Aligning with outgroups

* Review alignments and phylogenetic trees generated in `groups/family/` and `groups/group/`

* Add outgroups to `plastid_t_o.md` manually.

* *D* between target and outgroup should be around **0.05**.

* Locate targets of a family in mash groups

```bash
cd ~/data/plastid/

# family
cat taxon/Orchidaceae |
    parallel -j 1 -k 'rg -F -l {} taxon/group_*' |
    grep -v ".tsv" |
    sort |
    uniq |
    parallel -j 1 -k 'echo {}; cat {};'

```

* Check branch length between target and outgroup

```bash
cd ~/data/plastid/

cat ~/Scripts/withncbi/doc/plastid_t_o.md |
    grep -v "^#" |
    grep . |
    grep ",.*," | #head -n 35 |
    parallel -j 6 -k --col-sep "," '
        echo "==> {2}"
        files=$(
            rg -F -l {2} groups -g "*.nwk" -g "!fake*" -g "!*.raw.*" |
                xargs -I[] rg -F -l {3} []
            )

        for file in $files; do
            distance=$(
                cat ${file} |
                    nw_distance -m matrix -n - {2} {3} |
                    tsv-select -f 2 |
                    tsv-filter -H --ne 1:0 |
                    sed "1d"
                )

            # avoid
            if [ $(bc <<< "${distance} < 1") -eq 1 ]; then
                echo -e "{2}\t{3}\t${distance}\t${file}"
            fi
        done
    ' |
    tee /dev/stderr |
    grep -v "^==" > summary/outgroups.tsv

tsv-summarize summary/outgroups.tsv \
    --group-by 1,2 --count |
    wc -l

# no outgroups
cat ~/Scripts/withncbi/doc/plastid_t_o.md |
    head -n 674 | # Angiosperms
    grep -v "^#" |
    grep . |
    grep -v ",.*," | wc -l

tsv-summarize summary/outgroups.tsv \
    --mean 3 --min 3 --max 3

tsv-filter summary/outgroups.tsv \
    --le 3:0.01

tsv-filter summary/outgroups.tsv \
    --ge 3:0.1

```

* Find a suitable outgroup

```bash
cd ~/data/plastid/

parallel -j 1 -k '
    echo "==> {}"

    files=$(
        rg -F -l {} groups -g "*.nwk" -g "!fake*" -g "!*.raw.*"
        )

    for file in $files; do
        echo "    ${file}"
        cat ${file} |
            nw_distance -m matrix -n - ${TARGET} |
            sed "s/^\t/name\t/" |
            mlr --itsv --otsv cut -o -f "name,{}" |
            tsv-filter -H --le 2:0.07 --ge 2:0.03 |
            sed "1d"
    done
    ' ::: \
        Apos_odorata

```

```bash
cd ~/data/plastid/

# genus_og
cat taxon/group_target.tsv |
    tsv-filter -H --le 1:1000 |
    sed -e '1d' | #grep -w "^24" |
    parallel --colsep '\t' --no-run-if-empty --linebuffer -k -j 1 '
        outgroup=$(
            cat ~/Scripts/withncbi/doc/plastid_t_o.md |
                grep -v "^#" |
                grep . |
                perl -nl -e '\'' m/{2},{4},(\w+)/ and print $1 '\''
            )

        if [ "${outgroup}" = "" ]; then
            exit;
        fi

        if [ ! -d "GENOMES/${outgroup}" ]; then
            exit;
        fi

        echo -e "==> Group: [{2}]\tTarget: [{4}]\tOutgroup: [${outgroup}]\n"

        egaz template \
            GENOMES/{4} \
            $(cat taxon/{2} | grep -v -x "{4}" | xargs -I[] echo "GENOMES/[]") \
            GENOMES/${outgroup} \
            --multi -o groups/genus_og/{2}_og \
            --outgroup ${outgroup} \
            --taxon ~/data/organelle/plastid/GENOMES/taxon_ncbi.csv \
            --rawphylo --aligndb --parallel 4 -v

        bash groups/genus_og/{2}_og/1_pair.sh
        bash groups/genus_og/{2}_og/2_rawphylo.sh
        bash groups/genus_og/{2}_og/3_multi.sh
        bash groups/genus_og/{2}_og/6_chr_length.sh
        bash groups/genus_og/{2}_og/7_multi_aligndb.sh
    '

```

## Self alignments

```bash
cd ~/data/plastid/

cat taxon/group_target.tsv |
    tsv-filter -H --le 1:1000 |
    sed -e '1d' | grep -w "^24" |
    parallel --colsep '\t' --no-run-if-empty --linebuffer -k -j 6 '
        echo -e "==> Group: [{2}]\tTarget: [{4}]\n"

        egaz template \
            GENOMES/{4} \
            $(cat taxon/{2} | grep -v -x "{4}" | xargs -I[] echo "GENOMES/[]") \
            --self -o groups/self/{2} \
            --circos --parallel 4 -v

        bash groups/self/{2}/1_self.sh
        bash groups/self/{2}/3_proc.sh
        bash groups/self/{2}/4_circos.sh
    '

```

# LSC and SSC

IRA and IRB are presented by `plastid/self/${GENUS}/Results/${STRAIN}/${STRAIN}.links.tsv`.

```bash
find ~/data/plastid/self -type f -name "*.links.tsv" |
    xargs wc -l |
    sort -n |
    grep -v "total" |
    perl -nl -e 's/^\s*//g; /^(\d+)\s/ and print $1' |
    uniq -c |
    perl -nl -e '/^\s+(\d+)\s+(.+)$/ and print qq{$1\t$2}' |
    (echo -e 'count\tlines' && cat) |
    mlr --itsv --omd cat

```

Manually check strains not containing singular link.

| count | lines |
|:------|:------|
| 227   | 0     |
| 1949  | 1     |
| 44    | 2     |
| 22    | 3     |
| 4     | 4     |
| 2     | 10    |

| count | lines |
|------:|------:|
|   155 |     0 |
|  1709 |     1 |
|    36 |     2 |
|    17 |     3 |
|     1 |     4 |
|     1 |     5 |
|     1 |     8 |
|     1 |    10 |

There are 2 special strains (Asa_sieboldii, Epipo_aphyllum) which have no IR but a palindromic
sequence.

Create `ir_lsc_ssc.tsv` for slicing alignments.

Manually edit it then move to `~/Scripts/withncbi/doc/ir_lsc_ssc.tsv`.

`#genus abbr role accession chr_size IR LSC SSC`

* `NA` - not self-aligned
* `MULTI` - multiply link records
* `NONE` - no link records
* `WRONG` - unexpected link records

```bash
cd ~/data/plastid/summary

cat ABBR.csv |
    grep -v "^#" |
    perl -nla -F"," -MAlignDB::IntSpan -MPath::Tiny -e '
        BEGIN {
            %seen = ();
            @ls = grep {/\S/}
                  grep {!/^#/}
                  path(q{family.tsv})->lines({ chomp => 1});
            for (@ls) {
                $seen{$_}++ for split(/,|\s+/);
            }

            %target = ();
            %queries = ();
            @ls = grep {/\S/}
                  grep {!/^#/}
                  path(q{genus.tsv})->lines({ chomp => 1});
            for (@ls) {
                $target{(split /\t/)[1]}++;
                $queries{$_}++ for split(/,/, (split /\t/)[2]);
            }

            %outgroup = ();
            @ls = grep {/\S/}
                  grep {!/^#/}
                  path(q{genus_OG.tsv})->lines({ chomp => 1});
            for (@ls) {
                $outgroup{(split /\t/)[3]}++;
            }
        }

        chomp for @F;

        my $genus = $F[4];
        my $abbr = $F[9];

        next unless $seen{$abbr};

        my $role = "UNKNOWN";
        if ($target{$abbr}) {
            $role = "Target";
        }
        elsif ($queries{$abbr}) {
            $role = "Queries";
        }
        elsif ($outgroup{$abbr}) {
            $role = "Outgroup";
        }

        my $size_file = qq{$ENV{HOME}/data/plastid_self.working/$genus/Genomes/$abbr/chr.sizes};
        my $link_file = qq{$ENV{HOME}/data/plastid_self.working/$genus/Results/$abbr/$abbr.links.tsv};

        if (! -e $size_file or ! -e $link_file) {
            if ($outgroup{$abbr}) {
                print q{#} . join(qq{\t}, $genus, $abbr, $role, (q{NA}) x 5 );
            }
            next;
        }

        my $accession = `cat $size_file | cut -f 1 | xargs echo`;
        my $chr_size = `cat $size_file | cut -f 2 | xargs echo`;
        my $link = `cat $link_file | xargs echo`;
        chomp $accession; chomp $chr_size; chomp $link;

        if (split(q{ }, $chr_size) > 2 or split(q{ }, $link) > 2) {
            print q{#} . join(qq{\t}, $genus, $abbr, $role, $accession, $chr_size, (q{MULTI}) x 3 );
            next;
        }

        if ($link !~ m{:(\d+\-\d+)\s+.+:(\d+\-\d+)$}) {
            print q{#} . join(qq{\t}, $genus, $abbr, $role, $accession, $chr_size, (q{NONE}) x 3 );
            next;
        }

        my $ira = AlignDB::IntSpan->new($1);
        my $irb = AlignDB::IntSpan->new($2);
        my $ir = AlignDB::IntSpan->new->add($ira)->add($irb);

        if ($ira->trim(1)->contains(1)
            or $irb->trim(1)->contains(1)
            or $ira->trim(1)->contains($chr_size)
            or $irb->trim(1)->contains($chr_size)
            or ($ira->max + 1 > $irb->min - 1)) {
            print q{#} . join(qq{\t}, $genus, $abbr, $role, $accession, $chr_size, (q{WRONG}) x 3 );
            next;
        }

        my $chr = AlignDB::IntSpan->new->add_pair(1, $chr_size);
        my $interval = AlignDB::IntSpan->new->add_pair($ira->max + 1, $irb->min - 1);

        my $d_s_a = $ira->distance(AlignDB::IntSpan->new(1));
        my $d_b_e = $irb->distance(AlignDB::IntSpan->new($chr_size));
        my $d_a_b = $ira->distance($irb);
        $d_s_a = 0 if $d_s_a < 0;
        $d_b_e = 0 if $d_b_e < 0;

        my $lsc = AlignDB::IntSpan->new;
        my $ssc = AlignDB::IntSpan->new;

        if ($d_s_a + $d_b_e > $d_a_b) { # LSC
            $ssc = $interval->copy;
            $lsc = $chr->diff($ir)->diff($ssc);
        }
        else  { # SSC
            $lsc = $interval->copy;
            $ssc = $chr->diff($ir)->diff($lsc);
        }

        print join(qq{\t}, $genus, $abbr, $role, $accession, $chr_size, $ir, $lsc, $ssc );
    ' \
    > ir_lsc_ssc.tsv

```

## Can't get clear IR information

* Grateloupia
  * Grat_filicina
  * Grat_taiwanensis
* Caulerpa
  * Cau_cliftonii
  * Cau_racemosa
* Caloglossa
  * Calog_beccarii
  * Calog_intermedia
  * Calog_monosticha
* Pisum
  * Pisum_fulvum
  * Pisum_sativum
  * Pisum_sativum_subsp_elatius
* Dasya
  * Dasya_naccarioides
* Diplazium
  * Diplazium_unilobum
* Bryopsis
  * Bryop_plumosa
  * Bryop_sp_HV04063
  * Bry_sp_HV04063
* Medicago
  * Med_falcata
  * Med_hybrida
  * Med_papillosa
  * Med_truncatula
* Aegilops
  * Aeg_cylindrica
  * Aeg_geniculata
  * Aeg_speltoides
  * Aeg_tauschii
* Prototheca
  * Prot_cutis
  * Prot_stagnorum
  * Prot_zopfii
* Cryptomonas
  * Cryptomo_curvata
  * Cryptomo_paramecium
* Monotropa
  * Monotropa_hypopitys
* Liagora
  * Liagora_brachyclada
  * Liagora_harveyana
* Taiwania
  * Tai_cryptomerioides
  * Tai_flousiana
* Pinus
  * Pinus_armandii
  * Pinus_bungeana
  * Pinus_contorta
  * Pinus_gerardiana
  * Pinus_greggii
  * Pinus_jaliscana
  * Pinus_koraiensis
  * Pinus_krempfii
  * Pinus_lambertiana
  * Pinus_longaeva
  * Pinus_massoniana
  * Pinus_monophylla
  * Pinus_nelsonii
  * Pinus_oocarpa
  * Pinus_pinea
  * Pinus_sibirica
  * Pinus_strobus
  * Pinus_sylvestris
  * Pinus_tabuliformis
  * Pinus_taeda
  * Pinus_taiwanensis
  * Pinus_thunbergii
* Taxus
  * Taxus_mairei
* Picea
  * Pic_abies
  * Pic_asperata
  * Pic_crassifolia
  * Pic_glauca
  * Pic_jezoensis
  * Pic_morrisonicola
  * Pic_sitchensis
* Gracilariopsis
  * Gracilario_chorda
  * Gracilario_lemaneiformis
* Fragaria
  * Frag_mandshurica
  * Frag_vesca_subsp_bracteata
* Ulva
  * Ulva_fasciata
  * Ulva_flexuosa
  * Ulva_linza
  * Ulva_prolifera
* Monomorphina
  * Monom_parapyrum
* Epipogium
  * Epipo_aphyllum
* Euglena
  * Euglena_archaeoplastidiata
  * Euglena_viridis
* Chlorella
  * Chlore_heliozoae
  * Chlore_sorokiniana
  * Chlore_variabilis
  * Chlore_vulgaris
* Erodium
  * Ero_carvifolium
  * Ero_crassifolium
  * Ero_manescavi
  * Ero_rupestre
* Larix
  * Lar_decidua
  * Lar_sibirica
* Amentotaxus
  * Ame_argotaenia
  * Ame_formosana
* Pyropia
  * Pyro_perforata
* Ceramium
  * Ceram_cimbricum
  * Ceram_japonicum
  * Ceram_sungminbooi
* Hildenbrandia
  * Hilde_rivularis
  * Hilde_rubra
* Pilostyles
  * Pilo_aethiopica
  * Pilo_hamiltonii
* Codium
  * Codi_decorticatum
  * Codi_sp_arenicola
* Torreya
  * Torreya_fargesii
  * Torreya_grandis
* Vertebrata
  * Vert_australis
  * Vert_isogona
  * Vert_lanosa
  * Vert_thuyoides
* Wisteria
  * Wis_floribunda
  * Wis_sinensis
* Phelipanche
  * Pheli_purpurea
  * Pheli_ramosa
* Glycyrrhiza
  * Glycy_glabra
  * Glycy_lepidota
* Cephalotaxus
  * Cephalo_wilsoniana
* Polysiphonia
  * Polysi_brodiei
  * Polysi_elongata
  * Polysi_infestans
  * Polysi_schneideri
  * Polysi_scopulorum
  * Polysi_sertularioides
  * Polysi_stricta
* Lathyrus
  * Lathy_clymenum
  * Lathy_davidii
  * Lathy_graminifolius
  * Lathy_inconspicuus
  * Lathy_littoralis
  * Lathy_ochroleucus
  * Lathy_odoratus
  * Lathy_palustris
  * Lathy_sativus
  * Lathy_tingitanus
* Babesia
  * Bab_orientalis
* Gelidium
  * Gelidi_elegans
  * Gelidi_vagum
* Bostrychia
  * Bos_moritziana
  * Bos_simpliciuscula
  * Bos_tenella
* Membranoptera
  * Mem_platyphylla
  * Mem_tenuis
  * Mem_weeksiae
* Astragalus
  * Astra_mongholicus
  * Astra_mongholicus_var_nakaianus
* Asarum
  * Asa_minus
  * Asa_sieboldii
* Gracilaria
  * Gracilaria_changii
  * Gracilaria_chilensis
  * Gracilaria_firma
  * Gracilaria_salicornia
  * Gracilaria_tenuistipitata_var_liui
  * Gracilaria_vermiculophylla
* Plasmodium
  * Plas_chabaudi_chabaudi
  * Plas_falciparum_HB3
  * Plas_gallinaceum
  * Plas_relictum
  * Plas_vivax
* Triticum
  * Trit_urartu
* Trifolium
  * Trif_aureum
  * Trif_boissieri
  * Trif_glanduliferum
  * Trif_grandiflorum
  * Trif_strictum

There are 2 special strains which has only one palindromic sequence rather than IR. (as mentioned
before)

## Slices of IR, LSC and SSC

Without outgroups.

Be cautious to alignments with low coverage.

```bash
mkdir -p ~/data/plastid/slices
cd ~/data/plastid/slices

cat ~/Scripts/withncbi/doc/ir_lsc_ssc.tsv |
    perl -nla -F"\t" -MAlignDB::IntSpan -Mstrict -Mwarnings -e '
        /^#/ and next;
        $F[2] eq q{Target} or next;
        $F[5] =~ /\d+/ or next;

        print qq{# $F[0]};

        next unless -e "$ENV{HOME}/data/plastid/genus/$F[0]/$F[0]_refined/$F[3].synNet.maf.gz.fas.gz";

        my %rl_of = (
            IR  => $F[5],
            LSC => $F[6],
            SSC => $F[7],
        );

        my $lsc = AlignDB::IntSpan->new($F[6]);
        # next unless $lsc->span_size == 2; # for testing

        my $max_seg = 3;
        my $segment = int($lsc->size / $max_seg /2);
        for my $i (1 .. $max_seg) {
            my $slice_5 = AlignDB::IntSpan->new;
            my $slice_3 = AlignDB::IntSpan->new;

            # these are all indexes of LCS
            my $seg_5_start = $segment * ($i - 1) + 1;
            my $seg_5_end = $segment * $i;
            my $seg_3_start = $lsc->size - $segment * $i + 1;
            my $seg_3_end = $lsc->size - $segment * ($i - 1);

            if ($lsc->span_size == 1) {
                $slice_5 = $lsc->slice($seg_5_start, $seg_5_end);
                $slice_3 = $lsc->slice($seg_3_start, $seg_3_end);
            }
            elsif ($lsc->span_size == 2) { # start point inside LSC
                my ($lsc1, $lsc2) = $lsc->sets;

                if ($lsc2->size >= $seg_5_end) {
                    $slice_5 = $lsc2->slice($seg_5_start, $seg_5_end);
                }
                elsif ($lsc2->size >= $seg_5_start) {
                    $slice_5 = $lsc2->slice($seg_5_start, $lsc2->size);
                    $slice_5->add( $lsc1->slice(1, $segment - ($lsc2->size - $seg_5_start + 1) ) );
                }
                else {
                    $slice_5 = $lsc1->slice($seg_5_start - $lsc2->size, $seg_5_end - $lsc2->size);
                }

                if ($lsc1->size >= $lsc->size - $seg_3_start) {
                    $slice_3 = $lsc1->slice($seg_3_start - $lsc2->size, $seg_3_end - $lsc2->size);
                }
                elsif ($lsc1->size >= $lsc->size - $seg_3_end) {
                    $slice_3 = $lsc1->slice(1, $seg_3_end - $lsc2->size);
                    $slice_3->add( $lsc2->slice( $seg_3_start - ($seg_3_end - $lsc2->size), $lsc2->size ) );
                }
                else {
                    $slice_3 = $lsc2->slice($seg_3_start, $seg_3_end);
                }
            }

            my $slice = AlignDB::IntSpan->new;
            $slice->add($slice_5)->add($slice_3);
            $slice = $slice->fill(10); # fill small gaps in LSC3
            $rl_of{"LSC$i"} = $slice->as_string;
        }

        for my $key (sort keys %rl_of) {
            print qq{jrunlist cover <(echo $F[3]:$rl_of{$key}) -o $F[0].$key.yml};
            print qq{fasops slice -n $F[1] -o $F[0].$key.fas \\};
            print qq{    ~/data/plastid/genus/$F[0]/$F[0]_refined/$F[3].synNet.maf.gz.fas.gz \\};
            print qq{    $F[0].$key.yml};
            print qq{perl ~/Scripts/alignDB/alignDB.pl \\};
            print qq{    -d $F[0]_$key \\};
            print qq{    -da ~/data/plastid_slices/$F[0].$key.fas \\};
            print qq{    -a ~/data/plastid/genus/$F[0]/Stats/anno.yml \\};
            print qq{    -chr ~/data/plastid/genus/$F[0]/chr_length.csv \\};
            print qq{    --lt 1000 --parallel 8 --batch 5 \\};
            print qq{    --run common};
            print qq{};
        }

        print qq{};
    ' \
    > slices.sh

```

Run the generated bash file.

```bash
cd ~/data/plastid_slices

bash slices.sh
perl ~/Scripts/fig_table/collect_common_basic.pl -d .

```

# Summary

## Copy xlsx files

```bash
mkdir -p ~/data/plastid/summary/xlsx
cd ~/data/plastid/summary/xlsx

find ../../groups/genus -type f -name "*.common.xlsx" |
    grep -v "vs[A-Z]" |
    parallel 'cp {} .'

find ../../groups/genus_og -type f -name "*.common.xlsx" |
    grep -v "vs[A-Z]" |
    parallel 'cp {} .'

```

## Genome list

Create `list.csv` from `GENUS.csv` with sequence lengths.

```bash
mkdir -p ~/data/plastid/summary/table
cd ~/data/plastid/summary/table

# manually set orders in `plastid_OG.md`
perl -l -MPath::Tiny -e '
    BEGIN {
        @ls = map {/^#/ and s/^(#+\s*\w+).*/\1/; $_}
            map {s/,.+$//; $_}
            map {s/^###\s*//; $_}
            path(q{~/Scripts/withncbi/doc/plastid_t_o.md})->lines({chomp => 1});
    }
    for (@ls) {
        (/^\s*$/ or /^##\s+/ or /^#\s+(\w+)/) and next;
        print $_
    }
    ' \
    > genus_all.lst

# abbr accession length
find ../../groups/genus -type f -name "chr_length.csv" |
    parallel --jobs 1 --keep-order -r '
        perl -nl -e '\''
            BEGIN {
                our $l = { }; # avoid parallel replace string
            }

            next unless /\w+,\d+/;
            my ($common_name, undef, $chr, $length) = split /,/;
            if (exists $l->{$common_name}) {
                $l->{$common_name}{$chr} = $length;
            }
            else {
                $l->{$common_name} = {$chr => $length};
            }

            END {
                for my $common_name (keys %{$l}) {
                    my $chrs = join "|", sort keys %{$l->{$common_name}};
                    my $length = 0;
                    $length += $_ for values %{$l->{$common_name}};
                    print qq{$common_name\t$chrs\t$length}
                }
            }
        '\'' {}
    ' \
    > length.tmp
cat length.tmp | datamash check
#2248 lines, 3 fields

# phylum family genus abbr taxon_id
cat ~/data/plastid/summary/GENUS.csv |
    grep -v "^#" |
    perl -nla -F"," -e 'print join qq{\t}, ($F[8], $F[5], $F[4], $F[9], $F[0], )' |
    sort |
    uniq \
    > abbr.tmp
cat abbr.tmp | datamash check
#2250 lines, 5 fields

tsv-join \
    abbr.tmp \
    --data-fields 4 \
    -f length.tmp \
    --key-fields 1 \
    --append-fields 2,3 \
    > list.tmp
cat list.tmp | datamash check
#2248 lines, 7 fields

# sort as orders in plastid_t_o.md
echo -e "#phylum,family,genus,abbr,taxon_id,accession,length" > list.csv
cat list.tmp |
    perl -nl -a -MPath::Tiny -e '
        BEGIN{
            %genus;
            my @l1 = path(q{genus_all.lst})->lines({ chomp => 1});
            $genus{$l1[$_]} = $_ for (0 .. $#l1);
        }
        my $idx = $genus{$F[2]};
        die qq{$_\n} unless defined $idx;
        print qq{$_\t$idx};
    ' |
    sort -n -k8,8 |
    cut -f 1-7 |
    tr $'\t' ',' \
    >> list.csv
cat list.csv | datamash check -t,
#2249 lines, 7 fields

rm *.tmp

```

## Statistics of genome alignments

Some genera will be filtered out here.

Criteria:

* Coverage >= 0.5
* Total number of indels >= 100
* D of multiple alignments < 0.05

```bash
cd ~/data/plastid/summary/xlsx

cat <<'EOF' > Table_alignment.tt
---
autofit: A:F
texts:
  - text: "Genus"
    pos: A1
  - text: "No. of genomes"
    pos: B1
  - text: "Aligned length (Mb)"
    pos: C1
  - text: "Indels Per 100 bp"
    pos: D1
  - text: "D on average"
    pos: E1
  - text: "GC-content"
    pos: F1
[% FOREACH item IN data -%]
  - text: [% item.name %]
    pos: A[% loop.index + 2 %]
[% END -%]
borders:
  - range: A1:F1
    top: 1
  - range: A1:F1
    bottom: 1
ranges:
[% FOREACH item IN data -%]
  [% item.file %]:
    basic:
      - copy: B2
        paste: B[% loop.index + 2 %]
      - copy: B4
        paste: C[% loop.index + 2 %]
      - copy: B5
        paste: D[% loop.index + 2 %]
      - copy: B7
        paste: E[% loop.index + 2 %]
      - copy: B8
        paste: F[% loop.index + 2 %]
[% END -%]
EOF

cat ~/data/plastid/summary/table/genus_all.lst |
    grep -v "^#" |
    TT_FILE=Table_alignment.tt perl -MTemplate -nl -e '
        push @data, { name => $_, file => qq{$_.common.xlsx}, };
        END {
            $tt = Template->new;
            $tt->process($ENV{TT_FILE}, { data => \@data, })
                or die Template->error;
        }
    ' \
    > Table_alignment_all.yml

perl ~/Scripts/fig_table/xlsx_table.pl -i Table_alignment_all.yml
perl ~/Scripts/fig_table/xlsx2csv.pl -f Table_alignment_all.xlsx > Table_alignment_all.csv

cp -f Table_alignment_all.xlsx ~/data/plastid/summary/table
cp -f Table_alignment_all.csv ~/data/plastid/summary/table

```

```bash
cd ~/data/plastid/summary/table

echo "Genus,avg_size" > group_avg_size.csv
cat list.csv |
    grep -v "#" |
    perl -nla -F, -e '
        $count{$F[2]}++;
        $sum{$F[2]} += $F[6];
        END {
            for $k (sort keys %count) {
                printf qq{%s,%d\n}, $k, $sum{$k}/$count{$k};
            }
        }
    ' \
    >> group_avg_size.csv

cat Table_alignment_all.csv group_avg_size.csv |
    perl ~/Scripts/withncbi/util/merge_csv.pl -f 0 --concat -o stdout \
    > Table_alignment_all.1.csv

echo "Genus,coverage" > group_coverage.csv
cat Table_alignment_all.1.csv |
    perl -nla -F',' -e '
        $F[2] =~ /[\.\d]+/ or next;
        $F[6] =~ /[\.\d]+/ or next;
        $c = $F[2] * 1000 * 1000 / $F[6];
        print qq{$F[0],$c};
    ' \
    >> group_coverage.csv

cat Table_alignment_all.1.csv group_coverage.csv |
    perl ~/Scripts/withncbi/util/merge_csv.pl -f 0 --concat -o stdout \
    > Table_alignment_all.2.csv

echo "Genus,indels" > group_indels.csv
cat Table_alignment_all.2.csv |
    perl -nla -F',' -e '
        $F[6] =~ /[\.\d]+/ or next;
        $c = $F[3] / 100 * $F[2] * 1000 * 1000;
        print qq{$F[0],$c};
    ' \
    >> group_indels.csv

cat Table_alignment_all.2.csv group_indels.csv |
    perl ~/Scripts/withncbi/util/merge_csv.pl -f 0 --concat -o stdout |
    grep -v ',,' \
    > Table_alignment_for_filter.csv

# real filter
cat Table_alignment_for_filter.csv |
    perl -nla -F',' -e '
        $F[6] =~ /[\.\d]+/ or next;
        $F[0] =~ s/"//g;
        print $F[0] if ($F[7] >= 0.5 and $F[8] >= 100 and $F[4] < 0.05);
    ' \
    > genus.lst

rm ~/data/plastid/summary/table/Table_alignment_all.[0-9].csv
rm ~/data/plastid/summary/table/group_*csv

#
cd ~/data/plastid/summary/xlsx
cat ~/data/plastid/summary/table/genus.lst |
    grep -v "^#" |
    TT_FILE=Table_alignment.tt perl -MTemplate -nl -e '
        push @data, { name => $_, file => qq{$_.common.xlsx}, };
        END {
            $tt = Template->new;
            $tt->process($ENV{TT_FILE}, { data => \@data, })
                or die Template->error;
        }
    ' \
    > Table_alignment.yml

perl ~/Scripts/fig_table/xlsx_table.pl -i Table_alignment.yml
perl ~/Scripts/fig_table/xlsx2csv.pl -f Table_alignment.xlsx > Table_alignment.csv

cp -f ~/data/plastid/summary/xlsx/Table_alignment.xlsx ~/data/plastid/summary/table
cp -f ~/data/plastid/summary/xlsx/Table_alignment.csv ~/data/plastid/summary/table

```

## Groups

```bash
mkdir -p ~/data/plastid/summary/group
cd ~/data/plastid/summary/group

perl -l -MPath::Tiny -e '
    BEGIN {
        @ls = map {/^#/ and s/^(#+\s*\w+).*/\1/; $_}
            map {s/,\w+//; $_}
            map {s/^###\s*//; $_}
            path(q{~/Scripts/withncbi/doc/plastid_OG.md})->lines( { chomp => 1});
    }
    for (@ls) {
        (/^\s*$/ or /^##\s+/) and next;
        if (/^#\s+(\w+)/) {
            $fh = path("$1.txt")->openw;
            next;
        } else {
            print {$fh} $_;
        }
    }
    '

grep -Fx -f ../table/genus.lst Angiosperms.txt > Angiosperms.lst

find . -type f -name "*.txt" |
    xargs cat |
    grep -v -Fx -f Angiosperms.txt |
    grep -Fx -f ../table/genus.lst \
    > Others.lst

cat ../table/Table_alignment.csv |
    cut -d, -f 1,5 |
    perl -nla -F',' -e '$F[1] > 0.02 and $F[1] <= 0.05 and print $F[0];' |
    grep -Fx -f Angiosperms.lst \
    > group_3.lst

cat ../table/Table_alignment.csv |
    cut -d, -f 1,5 |
    perl -nla -F',' -e '$F[1] > 0.005 and $F[1] <= 0.02 and print $F[0];' |
    grep -Fx -f Angiosperms.lst \
    > group_2.lst

cat ../table/Table_alignment.csv |
    cut -d, -f 1,5 |
    perl -nla -F',' -e '$F[1] <= 0.005 and print $F[0];' |
    grep -Fx -f Angiosperms.lst \
    > group_1.lst

rm *.txt

```

NCBI Taxonomy tree

```bash
mkdir -p ~/data/plastid/summary/group
cd ~/data/plastid/summary/group

cat ~/data/plastid/summary/table/genus.lst |
    grep -v "^#" |
    perl -e '
        @ls = <>;
        $str = qq{bp_taxonomy2tree.pl \\\n};
        for (@ls) {
            chomp;
            $str .= qq{    -s "$_" \\\n};
        }
        $str .= qq{    -e \n};
        print $str;
    ' \
    > genera_tree.sh

bash genera_tree.sh > genera.newick

nw_display -s -b 'visibility:hidden' -w 600 -v 30 genera.newick |
    rsvg-convert -o genera.png

```

## Phylogenic trees of each genus with outgroup

```bash
mkdir -p ~/data/plastid/summary/trees
cd ~/data/plastid/summary/trees

cat ~/Scripts/withncbi/doc/plastid_t_o.md |
    grep -v "^#" |
    grep . |
    cut -d',' -f 1 \
    > list.txt

find ../../groups/genus_og -type f -path "*Results*" -name "*.nwk" |
    grep -v ".raw." |
    parallel -j 1 'cp {} .'

```

## d1, d2

`collect_xlsx.pl`

```bash
cd ~/data/plastid/summary/xlsx

cat <<'EOF' > cmd_collect_d1_d2.tt
perl ~/Scripts/fig_table/collect_xlsx.pl \
[% FOREACH item IN data -%]
    -f [% item.name %].common.xlsx \
    -s d1_pi_gc_cv \
    -n [% item.name %] \
[% END -%]
    -o cmd_d1.xlsx

perl ~/Scripts/fig_table/collect_xlsx.pl \
[% FOREACH item IN data -%]
    -f [% item.name %].common.xlsx \
    -s d2_pi_gc_cv \
    -n [% item.name %] \
[% END -%]
    -o cmd_d2.xlsx

perl ~/Scripts/fig_table/collect_xlsx.pl \
[% FOREACH item IN data -%]
    -f [% item.name %].common.xlsx \
    -s d1_comb_pi_gc_cv \
    -n [% item.name %] \
[% END -%]
    -o cmd_d1_comb.xlsx

perl ~/Scripts/fig_table/collect_xlsx.pl \
[% FOREACH item IN data -%]
    -f [% item.name %].common.xlsx \
    -s d2_comb_pi_gc_cv \
    -n [% item.name %] \
[% END -%]
    -o cmd_d2_comb.xlsx

EOF

cat ../table/genus.lst |
    grep -v "^#" |
    TT_FILE=cmd_collect_d1_d2.tt perl -MTemplate -nl -e '
        push @data, { name => $_, };
        END {
            $tt = Template->new;
            $tt->process($ENV{TT_FILE}, { data => \@data, })
                or die Template->error;
        }
    ' \
    > cmd_collect_d1_d2.sh

bash cmd_collect_d1_d2.sh

```

`sep_chart.pl`

```bash
mkdir -p ~/data/plastid/summary/fig

cd ~/data/plastid/summary/xlsx

cat <<'EOF' > cmd_chart_d1_d2.tt
perl ~/Scripts/fig_table/sep_chart.pl \
    -i cmd_d1.xlsx \
    -xl "Distance to indels ({italic(d)[1]})" \
    -yl "Nucleotide divergence ({italic(D)})" \
    -xr "A2:A8" -yr "B2:B8" \
    --y_min 0.0 --y_max [% y_max %] \
    -x_min 0 -x_max 5 \
    -rb "^([% FOREACH item IN data %][% item.name %]|[% END %]NON_EXIST)$" \
    -rs "NON_EXIST" \
    --postfix [% postfix %] --style_dot -ms

perl ~/Scripts/fig_table/sep_chart.pl \
    -i cmd_d1_comb.xlsx \
    -xl "Distance to indels ({italic(d)[1]})" \
    -yl "Nucleotide divergence ({italic(D)})" \
    -xr "A2:A8" -yr "B2:B8" \
    --y_min 0.0 --y_max [% y_max %] \
    -x_min 0 -x_max 5 \
    -rb "^([% FOREACH item IN data %][% item.name %]|[% END %]NON_EXIST)$" \
    -rs "NON_EXIST" \
    --postfix [% postfix %] --style_dot

perl ~/Scripts/fig_table/sep_chart.pl \
    -i cmd_d2.xlsx \
    -xl "Reciprocal of indel density ({italic(d)[2]})" \
    -yl "Nucleotide divergence ({italic(D)})" \
    -xr "A2:A23" -yr "B2:B23" \
    --y_min 0.0 --y_max [% y_max2 %] \
    -x_min 0 -x_max 20 \
    -rb "^([% FOREACH item IN data %][% item.name %]|[% END %]NON_EXIST)$" \
    -rs "NON_EXIST" \
    --postfix [% postfix %] --style_dot -ms

perl ~/Scripts/fig_table/sep_chart.pl \
    -i cmd_d2_comb.xlsx \
    -xl "Reciprocal of indel density ({italic(d)[2]})" \
    -yl "Nucleotide divergence ({italic(D)})" \
    -xr "A2:A23" -yr "B2:B23" \
    --y_min 0.0 --y_max [% y_max2 %] \
    -x_min 0 -x_max 20 \
    -rb "^([% FOREACH item IN data %][% item.name %]|[% END %]NON_EXIST)$" \
    -rs "NON_EXIST" \
    --postfix [% postfix %] --style_dot

EOF

cat ../group/group_1.lst |
    TT_FILE=cmd_chart_d1_d2.tt perl -MTemplate -nl -e '
        push @data, { name => $_, };
        END {
            $tt = Template->new;
            $tt->process($ENV{TT_FILE},
                { data => \@data,
                y_max => 0.01,
                y_max2 => 0.01,
                postfix => q{group_1}, })
                or die Template->error;
        }
    ' \
    > cmd_chart_group_1.sh

cat ../group/group_2.lst |
    TT_FILE=cmd_chart_d1_d2.tt perl -MTemplate -nl -e '
        push @data, { name => $_, };
        END {
            $tt = Template->new;
            $tt->process($ENV{TT_FILE},
                { data => \@data,
                y_max => 0.03,
                y_max2 => 0.03,
                postfix => q{group_2}, })
                or die Template->error;
        }
    ' \
    > cmd_chart_group_2.sh

cat ../group/group_3.lst |
    TT_FILE=cmd_chart_d1_d2.tt perl -MTemplate -nl -e '
        push @data, { name => $_, };
        END {
            $tt = Template->new;
            $tt->process($ENV{TT_FILE},
                { data => \@data,
                y_max => 0.05,
                y_max2 => 0.05,
                postfix => q{group_3}, })
                or die Template->error;
        }
    ' \
    > cmd_chart_group_3.sh

cat ../group/Others.lst |
    TT_FILE=cmd_chart_d1_d2.tt perl -MTemplate -nl -e '
        push @data, { name => $_, };
        END {
            $tt = Template->new;
            $tt->process($ENV{TT_FILE},
                { data => \@data,
                y_max => 0.15,
                y_max2 => 0.15,
                postfix => q{Others}, })
                or die Template->error;
        }
    ' \
    > cmd_chart_Others.sh

bash cmd_chart_group_1.sh
bash cmd_chart_group_2.sh
bash cmd_chart_group_3.sh
bash cmd_chart_Others.sh

rm ../xlsx/*.csv
mv ../xlsx/*.pdf ../fig

# Coreldraw doesn't play well with computer modern fonts (latex math).
# perl ~/Scripts/fig_table/tikz_chart.pl -i cmd_plastid_d1_A2A8_B2B8.group_1.csv -xl 'Distance to indels ($d_1$)' -yl 'Nucleotide divergence ($D$)' --y_min 0.0 --y_max 0.01 -x_min 0 -x_max 5 --style_dot --pdf

```

