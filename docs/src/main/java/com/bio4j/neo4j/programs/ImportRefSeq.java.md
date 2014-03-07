
 * Copyright (C) 2010-2013  "Bio4j"
 *
 * This file is part of Bio4j
 *
 * Bio4j is free software: you can redistribute it and/or modify
 * it under the terms of the GNU Affero General Public License as
 * published by the Free Software Foundation, either version 3 of the
 * License, or (at your option) any later version.
 * This program is distributed in the hope that it will be useful,
 * but WITHOUT ANY WARRANTY; without even the implied warranty of
 * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
 * GNU Affero General Public License for more details.
 * You should have received a copy of the GNU Affero General Public License
 * along with this program.  If not, see <http://www.gnu.org/licenses/>


```java
package com.bio4j.neo4jdb.programs;

import com.bio4j.neo4jdb.model.nodes.refseq.CDSNode;
import com.bio4j.neo4jdb.model.nodes.refseq.GeneNode;
import com.bio4j.neo4jdb.model.nodes.refseq.GenomeElementNode;
import com.bio4j.neo4jdb.model.nodes.refseq.rna.*;
import com.bio4j.neo4jdb.model.relationships.refseq.*;
import com.bio4j.neo4jdb.model.util.Bio4jManager;
import com.ohnosequences.util.Executable;
import com.ohnosequences.util.genbank.GBCommon;
import java.io.*;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.Map;
import java.util.logging.FileHandler;
import java.util.logging.Level;
import java.util.logging.Logger;
import java.util.logging.SimpleFormatter;
import org.neo4j.helpers.collection.MapUtil;
import org.neo4j.index.lucene.unsafe.batchinsert.LuceneBatchInserterIndexProvider;
import org.neo4j.unsafe.batchinsert.*;

/**
 * Imports RefSeq complete release into Bio4j
 *
 * @author Pablo Pareja Tobes <ppareja@era7.com>
 */
public class ImportRefSeq implements Executable {

    //--------indexing API constans-----
    private static String PROVIDER_ST = "provider";
    private static String EXACT_ST = "exact";
    private static String LUCENE_ST = "lucene";
    private static String TYPE_ST = "type";
    //-----------------------------------
    public static final String BASE_FOLDER = "refseq/release/complete/";
    private static final Logger logger = Logger.getLogger("ImportRefSeq");
    private static FileHandler fh;
    //------------------nodes properties maps-----------------------------------
    public static Map<String, Object> genomeElementProperties = new HashMap<String, Object>();
    public static Map<String, Object> geneProperties = new HashMap<String, Object>();
    public static Map<String, Object> cdsProperties = new HashMap<String, Object>();
    public static Map<String, Object> miscRnaProperties = new HashMap<String, Object>();
    public static Map<String, Object> mRnaProperties = new HashMap<String, Object>();
    public static Map<String, Object> ncRnaProperties = new HashMap<String, Object>();
    public static Map<String, Object> rRnaProperties = new HashMap<String, Object>();
    public static Map<String, Object> tmRnaProperties = new HashMap<String, Object>();
    public static Map<String, Object> tRnaProperties = new HashMap<String, Object>();
    //----------------------------------------------------------------------------------
    //--------------------------------relationships------------------------------------------
    public static GenomeElementGeneRel genomeElementGeneRel = new GenomeElementGeneRel(null);
    public static GenomeElementCDSRel genomeElementCDSRel = new GenomeElementCDSRel(null);
    public static GenomeElementMiscRnaRel genomeElementMiscRnaRel = new GenomeElementMiscRnaRel(null);
    public static GenomeElementMRnaRel genomeElementMRnaRel = new GenomeElementMRnaRel(null);
    public static GenomeElementNcRnaRel genomeElementNcRnaRel = new GenomeElementNcRnaRel(null);
    public static GenomeElementRRnaRel genomeElementRRnaRel = new GenomeElementRRnaRel(null);
    public static GenomeElementTmRnaRel genomeElementTmRnaRel = new GenomeElementTmRnaRel(null);
    public static GenomeElementTRnaRel genomeElementTRnaRel = new GenomeElementTRnaRel(null);
    //----------------------------------------------------------------------------------

    @Override
    public void execute(ArrayList<String> array) {
        String[] args = new String[array.size()];
        for (int i = 0; i < array.size(); i++) {
            args[i] = array.get(i);
        }
        main(args);
    }

    public static void main(String[] args) {


        if (args.length != 3) {
            System.out.println("This program expects the following parameters: \n"
                    + "1. Folder name with all the .gbk files \n"
                    + "2. Bio4j DB folder \n"
                    + "3. batch inserter .properties file");
        } else {

            long initTime = System.nanoTime();

            File inFolder = new File(args[0]);

            File[] files = inFolder.listFiles();

            BatchInserter inserter = null;
            BatchInserterIndexProvider indexProvider = null;


            //----------------------------------------------------------------------------------
            //---------------------initializing node type properties----------------------------
            genomeElementProperties.put(GenomeElementNode.NODE_TYPE_PROPERTY, GenomeElementNode.NODE_TYPE);
            geneProperties.put(GeneNode.NODE_TYPE_PROPERTY, GeneNode.NODE_TYPE);
            cdsProperties.put(CDSNode.NODE_TYPE_PROPERTY, CDSNode.NODE_TYPE);
            miscRnaProperties.put(MiscRNANode.NODE_TYPE_PROPERTY, MiscRNANode.NODE_TYPE);
            mRnaProperties.put(MRNANode.NODE_TYPE_PROPERTY, MRNANode.NODE_TYPE);
            ncRnaProperties.put(NcRNANode.NODE_TYPE_PROPERTY, NcRNANode.NODE_TYPE);
            rRnaProperties.put(RRNANode.NODE_TYPE_PROPERTY, RRNANode.NODE_TYPE);
            tmRnaProperties.put(TmRNANode.NODE_TYPE_PROPERTY, TmRNANode.NODE_TYPE);
            tRnaProperties.put(TRNANode.NODE_TYPE_PROPERTY, TRNANode.NODE_TYPE);
            //----------------------------------------------------------------------------------
            //----------------------------------------------------------------------------------


            BufferedWriter statsBuff = null;
            int genomeElementCounter = 0;

            try {
                // This block configures the logger with handler and formatter
                fh = new FileHandler("ImportRefSeq.log", false);

                SimpleFormatter formatter = new SimpleFormatter();
                fh.setFormatter(formatter);
                logger.addHandler(fh);
                logger.setLevel(Level.ALL);

                //---creating writer for stats file-----
                statsBuff = new BufferedWriter(new FileWriter(new File("ImportRefSeqStats.txt")));


                // create the batch inserter
                inserter = BatchInserters.inserter(args[1], MapUtil.load(new File(args[2])));

                // create the batch index service
                indexProvider = new LuceneBatchInserterIndexProvider(inserter);


                //-----------------create batch indexes----------------------------------
                //----------------------------------------------------------------------
                BatchInserterIndex genomeElementVersionIndex = indexProvider.nodeIndex(GenomeElementNode.GENOME_ELEMENT_VERSION_INDEX,
                        MapUtil.stringMap(PROVIDER_ST, LUCENE_ST, TYPE_ST, EXACT_ST));
                BatchInserterIndex nodeTypeIndex = indexProvider.nodeIndex(Bio4jManager.NODE_TYPE_INDEX_NAME,
                        MapUtil.stringMap(PROVIDER_ST, LUCENE_ST, TYPE_ST, EXACT_ST));


                for (File file : files) {
                    if (file.getName().endsWith(".gbff")) {

                        logger.log(Level.INFO, ("file: " + file.getName()));

                        BufferedReader reader = new BufferedReader(new FileReader(file));
                        String line;

                        while ((line = reader.readLine()) != null) {

                            //this is the first line where the locus is
                            String accessionSt = "";
                            String definitionSt = "";
                            String versionSt = "";
                            String commentSt = "";
                            StringBuilder seqStBuilder = new StringBuilder();

                            ArrayList<String> cdsList = new ArrayList<String>();
                            ArrayList<String> geneList = new ArrayList<String>();
                            ArrayList<String> miscRnaList = new ArrayList<String>();
                            ArrayList<String> mRnaList = new ArrayList<String>();
                            ArrayList<String> ncRnaList = new ArrayList<String>();
                            ArrayList<String> rRnaList = new ArrayList<String>();
                            ArrayList<String> tmRnaList = new ArrayList<String>();
                            ArrayList<String> tRnaList = new ArrayList<String>();

                            boolean originFound = false;

                            //Now I get all the lines till I reach the string '//'
                            do {
                                boolean readLineFlag = true;

                                if (line.startsWith(GBCommon.LOCUS_STR)) {
                                    // do nothing right now
                                } else if (line.startsWith(GBCommon.ACCESSION_STR)) {

                                    accessionSt = line.split(GBCommon.ACCESSION_STR)[1].trim();

                                } else if (line.startsWith(GBCommon.VERSION_STR)) {

                                    versionSt = line.split(GBCommon.VERSION_STR)[1].trim().split(" ")[0];

                                } else if (line.startsWith(GBCommon.DEFINITION_STR)) {

                                    definitionSt += line.split(GBCommon.DEFINITION_STR)[1].trim();
                                    do {
                                        line = reader.readLine();
                                        if (line.startsWith(" ")) {
                                            definitionSt += line.trim();
                                        }
                                    } while (line.startsWith(" "));
                                    readLineFlag = false;

                                } else if (line.startsWith(GBCommon.COMMENT_STR)) {

                                    commentSt += line.split(GBCommon.COMMENT_STR)[1].trim();
                                    do {
                                        line = reader.readLine();
                                        if (line.startsWith(" ")) {
                                            commentSt += "\n" + line.trim();
                                        }
                                    } while (line.startsWith(" "));
                                    readLineFlag = false;

                                } else if (line.startsWith(GBCommon.FEATURES_STR)) {


                                    do {
                                        line = reader.readLine();

                                        String lineSubstr5 = line.substring(5);

                                        if (lineSubstr5.startsWith(GBCommon.CDS_STR)) {
                                            String positionsSt = "";
                                            positionsSt += line.trim().split(GBCommon.CDS_STR)[1].trim();

                                            line = reader.readLine();

                                            while (!line.trim().startsWith("/")) {
                                                positionsSt += line.trim();
                                                line = reader.readLine();
                                            }

                                            cdsList.add(positionsSt);

                                        } else if (lineSubstr5.startsWith(GBCommon.GENE_STR)) {

                                            String positionsSt = "";
                                            positionsSt += line.trim().split(GBCommon.GENE_STR)[1].trim();

                                            line = reader.readLine();

                                            while (!line.trim().startsWith("/")) {
                                                positionsSt += line.trim();
                                                line = reader.readLine();
                                            }

                                            geneList.add(positionsSt);

                                        } else if (lineSubstr5.startsWith(GBCommon.MISC_RNA_STR)) {

                                            String positionsSt = "";
                                            positionsSt += line.trim().split(GBCommon.MISC_RNA_STR)[1].trim();

                                            line = reader.readLine();

                                            while (!line.trim().startsWith("/")) {
                                                positionsSt += line.trim();
                                                line = reader.readLine();
                                            }

                                            miscRnaList.add(positionsSt);

                                        } else if (lineSubstr5.startsWith(GBCommon.TM_RNA_STR)) {

                                            String positionsSt = "";
                                            positionsSt += line.trim().split(GBCommon.TM_RNA_STR)[1].trim();

                                            line = reader.readLine();

                                            while (!line.trim().startsWith("/")) {
                                                positionsSt += line.trim();
                                                line = reader.readLine();
                                            }

                                            tmRnaList.add(positionsSt);

                                        } else if (lineSubstr5.startsWith(GBCommon.R_RNA_STR)) {

                                            String positionsSt = "";
                                            positionsSt += line.trim().split(GBCommon.R_RNA_STR)[1].trim();

                                            line = reader.readLine();

                                            while (!line.trim().startsWith("/")) {
                                                positionsSt += line.trim();
                                                line = reader.readLine();
                                            }

                                            rRnaList.add(positionsSt);

                                        } else if (lineSubstr5.startsWith(GBCommon.M_RNA_STR)) {

                                            String positionsSt = "";
                                            positionsSt += line.trim().split(GBCommon.M_RNA_STR)[1].trim();

                                            line = reader.readLine();

                                            while (!line.trim().startsWith("/")) {
                                                positionsSt += line.trim();
                                                line = reader.readLine();
                                            }

                                            mRnaList.add(positionsSt);

                                        } else if (lineSubstr5.startsWith(GBCommon.NC_RNA_STR)) {

                                            String positionsSt = "";
                                            positionsSt += line.trim().split(GBCommon.NC_RNA_STR)[1].trim();

                                            line = reader.readLine();

                                            while (!line.trim().startsWith("/")) {
                                                positionsSt += line.trim();
                                                line = reader.readLine();
                                            }

                                            ncRnaList.add(positionsSt);

                                        } else if (lineSubstr5.startsWith(GBCommon.T_RNA_STR)) {

                                            String positionsSt = "";
                                            positionsSt += line.trim().split(GBCommon.T_RNA_STR)[1].trim();

                                            line = reader.readLine();

                                            while (!line.trim().startsWith("/")) {
                                                positionsSt += line.trim();
                                                line = reader.readLine();
                                            }

                                            tRnaList.add(positionsSt);

                                        }

                                    } while (line.startsWith(" "));
                                    readLineFlag = false;

                                } else if (line.startsWith(GBCommon.ORIGIN_STR)) {

                                    originFound = true;

                                    do {
                                        line = reader.readLine();
                                        String[] tempArray = line.trim().split(" ");
                                        for (int i = 1; i < tempArray.length; i++) {
                                            seqStBuilder.append(tempArray[i]);
                                        }

                                    } while (line.startsWith(" "));
                                    readLineFlag = false;
                                }

                                if (readLineFlag) {
                                    line = reader.readLine();
                                }



                            } while (line != null && !line.startsWith(GBCommon.LAST_LINE_STR));

                            //--------create genome element node--------------
                            long genomeElementId = createGenomeElementNode(versionSt,
                                    commentSt, definitionSt, inserter, genomeElementVersionIndex, nodeTypeIndex);

                            //-----------genes-----------------
                            for (String genePositionsSt : geneList) {
                                geneProperties.put(GeneNode.POSITIONS_PROPERTY, genePositionsSt);
                                long geneId = inserter.createNode(geneProperties);
                                inserter.createRelationship(genomeElementId, geneId, genomeElementGeneRel, null);

                                //indexing gene node by its node_type
                                nodeTypeIndex.add(geneId, MapUtil.map(Bio4jManager.NODE_TYPE_INDEX_NAME, GeneNode.NODE_TYPE));
                            }

                            //-----------CDS-----------------
                            for (String cdsPositionsSt : cdsList) {
                                cdsProperties.put(CDSNode.POSITIONS_PROPERTY, cdsPositionsSt);
                                long cdsID = inserter.createNode(cdsProperties);
                                inserter.createRelationship(genomeElementId, cdsID, genomeElementCDSRel, null);

                                //indexing CDS node by its node_type
                                nodeTypeIndex.add(cdsID, MapUtil.map(Bio4jManager.NODE_TYPE_INDEX_NAME, CDSNode.NODE_TYPE));
                            }

                            //-----------misc rna-----------------
                            for (String miscRnaPositionsSt : miscRnaList) {
                                miscRnaProperties.put(MiscRNANode.POSITIONS_PROPERTY, miscRnaPositionsSt);
                                long miscRnaID = inserter.createNode(miscRnaProperties);
                                inserter.createRelationship(genomeElementId, miscRnaID, genomeElementMiscRnaRel, null);

                                //indexing MiscRNA node by its node_type
                                nodeTypeIndex.add(miscRnaID, MapUtil.map(Bio4jManager.NODE_TYPE_INDEX_NAME, MiscRNANode.NODE_TYPE));
                            }

                            //-----------m rna-----------------
                            for (String mRnaPositionsSt : mRnaList) {
                                mRnaProperties.put(MRNANode.POSITIONS_PROPERTY, mRnaPositionsSt);
                                long mRnaID = inserter.createNode(mRnaProperties);
                                inserter.createRelationship(genomeElementId, mRnaID, genomeElementMRnaRel, null);

                                //indexing MRNA node by its node_type
                                nodeTypeIndex.add(mRnaID, MapUtil.map(Bio4jManager.NODE_TYPE_INDEX_NAME, MRNANode.NODE_TYPE));
                            }

                            //-----------nc rna-----------------
                            for (String ncRnaPositionsSt : ncRnaList) {
                                ncRnaProperties.put(NcRNANode.POSITIONS_PROPERTY, ncRnaPositionsSt);
                                long ncRnaID = inserter.createNode(ncRnaProperties);
                                inserter.createRelationship(genomeElementId, ncRnaID, genomeElementNcRnaRel, null);

                                //indexing NCRNA node by its node_type
                                nodeTypeIndex.add(ncRnaID, MapUtil.map(Bio4jManager.NODE_TYPE_INDEX_NAME, NcRNANode.NODE_TYPE));
                            }

                            //-----------r rna-----------------
                            for (String rRnaPositionsSt : rRnaList) {
                                rRnaProperties.put(RRNANode.POSITIONS_PROPERTY, rRnaPositionsSt);
                                long rRnaID = inserter.createNode(rRnaProperties);
                                inserter.createRelationship(genomeElementId, rRnaID, genomeElementRRnaRel, null);

                                //indexing RRNA node by its node_type
                                nodeTypeIndex.add(rRnaID, MapUtil.map(Bio4jManager.NODE_TYPE_INDEX_NAME, RRNANode.NODE_TYPE));
                            }

                            //-----------tm rna-----------------
                            for (String tmRnaPositionsSt : tmRnaList) {
                                tmRnaProperties.put(TmRNANode.POSITIONS_PROPERTY, tmRnaPositionsSt);
                                long tmRnaID = inserter.createNode(tmRnaProperties);
                                inserter.createRelationship(genomeElementId, tmRnaID, genomeElementTmRnaRel, null);

                                //indexing TmRNA node by its node_type
                                nodeTypeIndex.add(tmRnaID, MapUtil.map(Bio4jManager.NODE_TYPE_INDEX_NAME, TmRNANode.NODE_TYPE));
                            }

                            //-----------t rna-----------------
                            for (String tRnaPositionsSt : tRnaList) {
                                tRnaProperties.put(TRNANode.POSITIONS_PROPERTY, tRnaPositionsSt);
                                long tRnaID = inserter.createNode(tRnaProperties);
                                inserter.createRelationship(genomeElementId, tRnaID, genomeElementTRnaRel, null);

                                //indexing TRNA node by its node_type
                                nodeTypeIndex.add(tRnaID, MapUtil.map(Bio4jManager.NODE_TYPE_INDEX_NAME, TRNANode.NODE_TYPE));
                            }

                            logger.log(Level.INFO, (versionSt + " saved!"));

                            genomeElementCounter++;

                        }
                        
                        reader.close();

                    }
                }

            } catch (Exception e) {
                logger.log(Level.SEVERE, e.getMessage());
                StackTraceElement[] trace = e.getStackTrace();
                for (StackTraceElement stackTraceElement : trace) {
                    logger.log(Level.SEVERE, stackTraceElement.toString());
                }
            } finally {

                // shutdown, makes sure all changes are written to disk
                indexProvider.shutdown();
                inserter.shutdown();

                try {
                    //-----------------writing stats file---------------------
                    long elapsedTime = System.nanoTime() - initTime;
                    long elapsedSeconds = Math.round((elapsedTime / 1000000000.0));
                    long hours = elapsedSeconds / 3600;
                    long minutes = (elapsedSeconds % 3600) / 60;
                    long seconds = (elapsedSeconds % 3600) % 60;

                    statsBuff.write("Statistics for program ImportRefSeq:\nInput folder: " + inFolder.getName()
                            + "\nThere were " + genomeElementCounter + " genome elements stored.\n"
                            + "The elapsed time was: " + hours + "h " + minutes + "m " + seconds + "s\n");

                    //---closing stats writer---
                    statsBuff.close();
                } catch (Exception e) {
                    e.printStackTrace();
                }

                // closing logger file handler
                fh.close();

            }
        }


    }

    private static long createGenomeElementNode(String version,
            String comment,
            String definition,
            BatchInserter inserter,
            BatchInserterIndex genomeElementVersionIndex,
            BatchInserterIndex nodeTypeIndex) {

        genomeElementProperties.put(GenomeElementNode.VERSION_PROPERTY, version);
        genomeElementProperties.put(GenomeElementNode.COMMENT_PROPERTY, comment);
        genomeElementProperties.put(GenomeElementNode.DEFINITION_PROPERTY, definition);

        long genomeElementId = inserter.createNode(genomeElementProperties);
        genomeElementVersionIndex.add(genomeElementId, MapUtil.map(GenomeElementNode.GENOME_ELEMENT_VERSION_INDEX, version));
        nodeTypeIndex.add(genomeElementId, MapUtil.map(Bio4jManager.NODE_TYPE_INDEX_NAME, GenomeElementNode.NODE_TYPE));

        return genomeElementId;

    }
}

```


------

### Index

+ src
  + main
    + java
      + com
        + bio4j
          + neo4j
            + [BasicEntity.java][main/java/com/bio4j/neo4j/BasicEntity.java]
            + [BasicRelationship.java][main/java/com/bio4j/neo4j/BasicRelationship.java]
            + codesamples
              + [BiodieselProductionSample.java][main/java/com/bio4j/neo4j/codesamples/BiodieselProductionSample.java]
              + [GetEnzymeData.java][main/java/com/bio4j/neo4j/codesamples/GetEnzymeData.java]
              + [GetGenesInfo.java][main/java/com/bio4j/neo4j/codesamples/GetGenesInfo.java]
              + [GetGOAnnotationsForOrganism.java][main/java/com/bio4j/neo4j/codesamples/GetGOAnnotationsForOrganism.java]
              + [GetProteinsWithInterpro.java][main/java/com/bio4j/neo4j/codesamples/GetProteinsWithInterpro.java]
              + [RealUseCase1.java][main/java/com/bio4j/neo4j/codesamples/RealUseCase1.java]
              + [RetrieveProteinSample.java][main/java/com/bio4j/neo4j/codesamples/RetrieveProteinSample.java]
            + model
              + nodes
                + [AlternativeProductNode.java][main/java/com/bio4j/neo4j/model/nodes/AlternativeProductNode.java]
                + citation
                  + [ArticleNode.java][main/java/com/bio4j/neo4j/model/nodes/citation/ArticleNode.java]
                  + [BookNode.java][main/java/com/bio4j/neo4j/model/nodes/citation/BookNode.java]
                  + [DBNode.java][main/java/com/bio4j/neo4j/model/nodes/citation/DBNode.java]
                  + [JournalNode.java][main/java/com/bio4j/neo4j/model/nodes/citation/JournalNode.java]
                  + [OnlineArticleNode.java][main/java/com/bio4j/neo4j/model/nodes/citation/OnlineArticleNode.java]
                  + [OnlineJournalNode.java][main/java/com/bio4j/neo4j/model/nodes/citation/OnlineJournalNode.java]
                  + [PatentNode.java][main/java/com/bio4j/neo4j/model/nodes/citation/PatentNode.java]
                  + [PublisherNode.java][main/java/com/bio4j/neo4j/model/nodes/citation/PublisherNode.java]
                  + [SubmissionNode.java][main/java/com/bio4j/neo4j/model/nodes/citation/SubmissionNode.java]
                  + [ThesisNode.java][main/java/com/bio4j/neo4j/model/nodes/citation/ThesisNode.java]
                  + [UnpublishedObservationNode.java][main/java/com/bio4j/neo4j/model/nodes/citation/UnpublishedObservationNode.java]
                + [CityNode.java][main/java/com/bio4j/neo4j/model/nodes/CityNode.java]
                + [CommentTypeNode.java][main/java/com/bio4j/neo4j/model/nodes/CommentTypeNode.java]
                + [ConsortiumNode.java][main/java/com/bio4j/neo4j/model/nodes/ConsortiumNode.java]
                + [CountryNode.java][main/java/com/bio4j/neo4j/model/nodes/CountryNode.java]
                + [DatasetNode.java][main/java/com/bio4j/neo4j/model/nodes/DatasetNode.java]
                + [EnzymeNode.java][main/java/com/bio4j/neo4j/model/nodes/EnzymeNode.java]
                + [FeatureTypeNode.java][main/java/com/bio4j/neo4j/model/nodes/FeatureTypeNode.java]
                + [GoTermNode.java][main/java/com/bio4j/neo4j/model/nodes/GoTermNode.java]
                + [InstituteNode.java][main/java/com/bio4j/neo4j/model/nodes/InstituteNode.java]
                + [InterproNode.java][main/java/com/bio4j/neo4j/model/nodes/InterproNode.java]
                + [IsoformNode.java][main/java/com/bio4j/neo4j/model/nodes/IsoformNode.java]
                + [KeywordNode.java][main/java/com/bio4j/neo4j/model/nodes/KeywordNode.java]
                + ncbi
                  + [NCBITaxonNode.java][main/java/com/bio4j/neo4j/model/nodes/ncbi/NCBITaxonNode.java]
                + [OrganismNode.java][main/java/com/bio4j/neo4j/model/nodes/OrganismNode.java]
                + [PersonNode.java][main/java/com/bio4j/neo4j/model/nodes/PersonNode.java]
                + [PfamNode.java][main/java/com/bio4j/neo4j/model/nodes/PfamNode.java]
                + [ProteinNode.java][main/java/com/bio4j/neo4j/model/nodes/ProteinNode.java]
                + reactome
                  + [ReactomeTermNode.java][main/java/com/bio4j/neo4j/model/nodes/reactome/ReactomeTermNode.java]
                + refseq
                  + [CDSNode.java][main/java/com/bio4j/neo4j/model/nodes/refseq/CDSNode.java]
                  + [GeneNode.java][main/java/com/bio4j/neo4j/model/nodes/refseq/GeneNode.java]
                  + [GenomeElementNode.java][main/java/com/bio4j/neo4j/model/nodes/refseq/GenomeElementNode.java]
                  + rna
                    + [MiscRNANode.java][main/java/com/bio4j/neo4j/model/nodes/refseq/rna/MiscRNANode.java]
                    + [MRNANode.java][main/java/com/bio4j/neo4j/model/nodes/refseq/rna/MRNANode.java]
                    + [NcRNANode.java][main/java/com/bio4j/neo4j/model/nodes/refseq/rna/NcRNANode.java]
                    + [RNANode.java][main/java/com/bio4j/neo4j/model/nodes/refseq/rna/RNANode.java]
                    + [RRNANode.java][main/java/com/bio4j/neo4j/model/nodes/refseq/rna/RRNANode.java]
                    + [TmRNANode.java][main/java/com/bio4j/neo4j/model/nodes/refseq/rna/TmRNANode.java]
                    + [TRNANode.java][main/java/com/bio4j/neo4j/model/nodes/refseq/rna/TRNANode.java]
                + [SequenceCautionNode.java][main/java/com/bio4j/neo4j/model/nodes/SequenceCautionNode.java]
                + [SubcellularLocationNode.java][main/java/com/bio4j/neo4j/model/nodes/SubcellularLocationNode.java]
                + [TaxonNode.java][main/java/com/bio4j/neo4j/model/nodes/TaxonNode.java]
              + relationships
                + aproducts
                  + [AlternativeProductInitiationRel.java][main/java/com/bio4j/neo4j/model/relationships/aproducts/AlternativeProductInitiationRel.java]
                  + [AlternativeProductPromoterRel.java][main/java/com/bio4j/neo4j/model/relationships/aproducts/AlternativeProductPromoterRel.java]
                  + [AlternativeProductRibosomalFrameshiftingRel.java][main/java/com/bio4j/neo4j/model/relationships/aproducts/AlternativeProductRibosomalFrameshiftingRel.java]
                  + [AlternativeProductSplicingRel.java][main/java/com/bio4j/neo4j/model/relationships/aproducts/AlternativeProductSplicingRel.java]
                + citation
                  + article
                    + [ArticleAuthorRel.java][main/java/com/bio4j/neo4j/model/relationships/citation/article/ArticleAuthorRel.java]
                    + [ArticleJournalRel.java][main/java/com/bio4j/neo4j/model/relationships/citation/article/ArticleJournalRel.java]
                    + [ArticleProteinCitationRel.java][main/java/com/bio4j/neo4j/model/relationships/citation/article/ArticleProteinCitationRel.java]
                  + book
                    + [BookAuthorRel.java][main/java/com/bio4j/neo4j/model/relationships/citation/book/BookAuthorRel.java]
                    + [BookCityRel.java][main/java/com/bio4j/neo4j/model/relationships/citation/book/BookCityRel.java]
                    + [BookEditorRel.java][main/java/com/bio4j/neo4j/model/relationships/citation/book/BookEditorRel.java]
                    + [BookProteinCitationRel.java][main/java/com/bio4j/neo4j/model/relationships/citation/book/BookProteinCitationRel.java]
                    + [BookPublisherRel.java][main/java/com/bio4j/neo4j/model/relationships/citation/book/BookPublisherRel.java]
                  + onarticle
                    + [OnlineArticleAuthorRel.java][main/java/com/bio4j/neo4j/model/relationships/citation/onarticle/OnlineArticleAuthorRel.java]
                    + [OnlineArticleJournalRel.java][main/java/com/bio4j/neo4j/model/relationships/citation/onarticle/OnlineArticleJournalRel.java]
                    + [OnlineArticleProteinCitationRel.java][main/java/com/bio4j/neo4j/model/relationships/citation/onarticle/OnlineArticleProteinCitationRel.java]
                  + patent
                    + [PatentAuthorRel.java][main/java/com/bio4j/neo4j/model/relationships/citation/patent/PatentAuthorRel.java]
                    + [PatentProteinCitationRel.java][main/java/com/bio4j/neo4j/model/relationships/citation/patent/PatentProteinCitationRel.java]
                  + submission
                    + [SubmissionAuthorRel.java][main/java/com/bio4j/neo4j/model/relationships/citation/submission/SubmissionAuthorRel.java]
                    + [SubmissionDbRel.java][main/java/com/bio4j/neo4j/model/relationships/citation/submission/SubmissionDbRel.java]
                    + [SubmissionProteinCitationRel.java][main/java/com/bio4j/neo4j/model/relationships/citation/submission/SubmissionProteinCitationRel.java]
                  + thesis
                    + [ThesisAuthorRel.java][main/java/com/bio4j/neo4j/model/relationships/citation/thesis/ThesisAuthorRel.java]
                    + [ThesisInstituteRel.java][main/java/com/bio4j/neo4j/model/relationships/citation/thesis/ThesisInstituteRel.java]
                    + [ThesisProteinCitationRel.java][main/java/com/bio4j/neo4j/model/relationships/citation/thesis/ThesisProteinCitationRel.java]
                  + uo
                    + [UnpublishedObservationAuthorRel.java][main/java/com/bio4j/neo4j/model/relationships/citation/uo/UnpublishedObservationAuthorRel.java]
                    + [UnpublishedObservationProteinCitationRel.java][main/java/com/bio4j/neo4j/model/relationships/citation/uo/UnpublishedObservationProteinCitationRel.java]
                + comment
                  + [AllergenCommentRel.java][main/java/com/bio4j/neo4j/model/relationships/comment/AllergenCommentRel.java]
                  + [BasicCommentRel.java][main/java/com/bio4j/neo4j/model/relationships/comment/BasicCommentRel.java]
                  + [BioPhysicoChemicalPropertiesCommentRel.java][main/java/com/bio4j/neo4j/model/relationships/comment/BioPhysicoChemicalPropertiesCommentRel.java]
                  + [BiotechnologyCommentRel.java][main/java/com/bio4j/neo4j/model/relationships/comment/BiotechnologyCommentRel.java]
                  + [CatalyticActivityCommentRel.java][main/java/com/bio4j/neo4j/model/relationships/comment/CatalyticActivityCommentRel.java]
                  + [CautionCommentRel.java][main/java/com/bio4j/neo4j/model/relationships/comment/CautionCommentRel.java]
                  + [CofactorCommentRel.java][main/java/com/bio4j/neo4j/model/relationships/comment/CofactorCommentRel.java]
                  + [DevelopmentalStageCommentRel.java][main/java/com/bio4j/neo4j/model/relationships/comment/DevelopmentalStageCommentRel.java]
                  + [DiseaseCommentRel.java][main/java/com/bio4j/neo4j/model/relationships/comment/DiseaseCommentRel.java]
                  + [DisruptionPhenotypeCommentRel.java][main/java/com/bio4j/neo4j/model/relationships/comment/DisruptionPhenotypeCommentRel.java]
                  + [DomainCommentRel.java][main/java/com/bio4j/neo4j/model/relationships/comment/DomainCommentRel.java]
                  + [EnzymeRegulationCommentRel.java][main/java/com/bio4j/neo4j/model/relationships/comment/EnzymeRegulationCommentRel.java]
                  + [FunctionCommentRel.java][main/java/com/bio4j/neo4j/model/relationships/comment/FunctionCommentRel.java]
                  + [InductionCommentRel.java][main/java/com/bio4j/neo4j/model/relationships/comment/InductionCommentRel.java]
                  + [MassSpectrometryCommentRel.java][main/java/com/bio4j/neo4j/model/relationships/comment/MassSpectrometryCommentRel.java]
                  + [MiscellaneousCommentRel.java][main/java/com/bio4j/neo4j/model/relationships/comment/MiscellaneousCommentRel.java]
                  + [OnlineInformationCommentRel.java][main/java/com/bio4j/neo4j/model/relationships/comment/OnlineInformationCommentRel.java]
                  + [PathwayCommentRel.java][main/java/com/bio4j/neo4j/model/relationships/comment/PathwayCommentRel.java]
                  + [PharmaceuticalCommentRel.java][main/java/com/bio4j/neo4j/model/relationships/comment/PharmaceuticalCommentRel.java]
                  + [PolymorphismCommentRel.java][main/java/com/bio4j/neo4j/model/relationships/comment/PolymorphismCommentRel.java]
                  + [PostTranslationalModificationCommentRel.java][main/java/com/bio4j/neo4j/model/relationships/comment/PostTranslationalModificationCommentRel.java]
                  + [RnaEditingCommentRel.java][main/java/com/bio4j/neo4j/model/relationships/comment/RnaEditingCommentRel.java]
                  + [SimilarityCommentRel.java][main/java/com/bio4j/neo4j/model/relationships/comment/SimilarityCommentRel.java]
                  + [SubunitCommentRel.java][main/java/com/bio4j/neo4j/model/relationships/comment/SubunitCommentRel.java]
                  + [TissueSpecificityCommentRel.java][main/java/com/bio4j/neo4j/model/relationships/comment/TissueSpecificityCommentRel.java]
                  + [ToxicDoseCommentRel.java][main/java/com/bio4j/neo4j/model/relationships/comment/ToxicDoseCommentRel.java]
                + features
                  + [ActiveSiteFeatureRel.java][main/java/com/bio4j/neo4j/model/relationships/features/ActiveSiteFeatureRel.java]
                  + [BasicFeatureRel.java][main/java/com/bio4j/neo4j/model/relationships/features/BasicFeatureRel.java]
                  + [BindingSiteFeatureRel.java][main/java/com/bio4j/neo4j/model/relationships/features/BindingSiteFeatureRel.java]
                  + [CalciumBindingRegionFeatureRel.java][main/java/com/bio4j/neo4j/model/relationships/features/CalciumBindingRegionFeatureRel.java]
                  + [ChainFeatureRel.java][main/java/com/bio4j/neo4j/model/relationships/features/ChainFeatureRel.java]
                  + [CoiledCoilRegionFeatureRel.java][main/java/com/bio4j/neo4j/model/relationships/features/CoiledCoilRegionFeatureRel.java]
                  + [CompositionallyBiasedRegionFeatureRel.java][main/java/com/bio4j/neo4j/model/relationships/features/CompositionallyBiasedRegionFeatureRel.java]
                  + [CrossLinkFeatureRel.java][main/java/com/bio4j/neo4j/model/relationships/features/CrossLinkFeatureRel.java]
                  + [DisulfideBondFeatureRel.java][main/java/com/bio4j/neo4j/model/relationships/features/DisulfideBondFeatureRel.java]
                  + [DnaBindingRegionFeatureRel.java][main/java/com/bio4j/neo4j/model/relationships/features/DnaBindingRegionFeatureRel.java]
                  + [DomainFeatureRel.java][main/java/com/bio4j/neo4j/model/relationships/features/DomainFeatureRel.java]
                  + [GlycosylationSiteFeatureRel.java][main/java/com/bio4j/neo4j/model/relationships/features/GlycosylationSiteFeatureRel.java]
                  + [HelixFeatureRel.java][main/java/com/bio4j/neo4j/model/relationships/features/HelixFeatureRel.java]
                  + [InitiatorMethionineFeatureRel.java][main/java/com/bio4j/neo4j/model/relationships/features/InitiatorMethionineFeatureRel.java]
                  + [IntramembraneRegionFeatureRel.java][main/java/com/bio4j/neo4j/model/relationships/features/IntramembraneRegionFeatureRel.java]
                  + [LipidMoietyBindingRegionFeatureRel.java][main/java/com/bio4j/neo4j/model/relationships/features/LipidMoietyBindingRegionFeatureRel.java]
                  + [MetalIonBindingSiteFeatureRel.java][main/java/com/bio4j/neo4j/model/relationships/features/MetalIonBindingSiteFeatureRel.java]
                  + [ModifiedResidueFeatureRel.java][main/java/com/bio4j/neo4j/model/relationships/features/ModifiedResidueFeatureRel.java]
                  + [MutagenesisSiteFeatureRel.java][main/java/com/bio4j/neo4j/model/relationships/features/MutagenesisSiteFeatureRel.java]
                  + [NonConsecutiveResiduesFeatureRel.java][main/java/com/bio4j/neo4j/model/relationships/features/NonConsecutiveResiduesFeatureRel.java]
                  + [NonStandardAminoAcidFeatureRel.java][main/java/com/bio4j/neo4j/model/relationships/features/NonStandardAminoAcidFeatureRel.java]
                  + [NonTerminalResidueFeatureRel.java][main/java/com/bio4j/neo4j/model/relationships/features/NonTerminalResidueFeatureRel.java]
                  + [NucleotidePhosphateBindingRegionFeatureRel.java][main/java/com/bio4j/neo4j/model/relationships/features/NucleotidePhosphateBindingRegionFeatureRel.java]
                  + [PeptideFeatureRel.java][main/java/com/bio4j/neo4j/model/relationships/features/PeptideFeatureRel.java]
                  + [PropeptideFeatureRel.java][main/java/com/bio4j/neo4j/model/relationships/features/PropeptideFeatureRel.java]
                  + [RegionOfInterestFeatureRel.java][main/java/com/bio4j/neo4j/model/relationships/features/RegionOfInterestFeatureRel.java]
                  + [RepeatFeatureRel.java][main/java/com/bio4j/neo4j/model/relationships/features/RepeatFeatureRel.java]
                  + [SequenceConflictFeatureRel.java][main/java/com/bio4j/neo4j/model/relationships/features/SequenceConflictFeatureRel.java]
                  + [SequenceVariantFeatureRel.java][main/java/com/bio4j/neo4j/model/relationships/features/SequenceVariantFeatureRel.java]
                  + [ShortSequenceMotifFeatureRel.java][main/java/com/bio4j/neo4j/model/relationships/features/ShortSequenceMotifFeatureRel.java]
                  + [SignalPeptideFeatureRel.java][main/java/com/bio4j/neo4j/model/relationships/features/SignalPeptideFeatureRel.java]
                  + [SiteFeatureRel.java][main/java/com/bio4j/neo4j/model/relationships/features/SiteFeatureRel.java]
                  + [SpliceVariantFeatureRel.java][main/java/com/bio4j/neo4j/model/relationships/features/SpliceVariantFeatureRel.java]
                  + [StrandFeatureRel.java][main/java/com/bio4j/neo4j/model/relationships/features/StrandFeatureRel.java]
                  + [TopologicalDomainFeatureRel.java][main/java/com/bio4j/neo4j/model/relationships/features/TopologicalDomainFeatureRel.java]
                  + [TransitPeptideFeatureRel.java][main/java/com/bio4j/neo4j/model/relationships/features/TransitPeptideFeatureRel.java]
                  + [TransmembraneRegionFeatureRel.java][main/java/com/bio4j/neo4j/model/relationships/features/TransmembraneRegionFeatureRel.java]
                  + [TurnFeatureRel.java][main/java/com/bio4j/neo4j/model/relationships/features/TurnFeatureRel.java]
                  + [UnsureResidueFeatureRel.java][main/java/com/bio4j/neo4j/model/relationships/features/UnsureResidueFeatureRel.java]
                  + [ZincFingerRegionFeatureRel.java][main/java/com/bio4j/neo4j/model/relationships/features/ZincFingerRegionFeatureRel.java]
                + go
                  + [BiologicalProcessRel.java][main/java/com/bio4j/neo4j/model/relationships/go/BiologicalProcessRel.java]
                  + [CellularComponentRel.java][main/java/com/bio4j/neo4j/model/relationships/go/CellularComponentRel.java]
                  + [HasPartOfGoRel.java][main/java/com/bio4j/neo4j/model/relationships/go/HasPartOfGoRel.java]
                  + [IsAGoRel.java][main/java/com/bio4j/neo4j/model/relationships/go/IsAGoRel.java]
                  + [MainGoRel.java][main/java/com/bio4j/neo4j/model/relationships/go/MainGoRel.java]
                  + [MolecularFunctionRel.java][main/java/com/bio4j/neo4j/model/relationships/go/MolecularFunctionRel.java]
                  + [NegativelyRegulatesGoRel.java][main/java/com/bio4j/neo4j/model/relationships/go/NegativelyRegulatesGoRel.java]
                  + [PartOfGoRel.java][main/java/com/bio4j/neo4j/model/relationships/go/PartOfGoRel.java]
                  + [PositivelyRegulatesGoRel.java][main/java/com/bio4j/neo4j/model/relationships/go/PositivelyRegulatesGoRel.java]
                  + [RegulatesGoRel.java][main/java/com/bio4j/neo4j/model/relationships/go/RegulatesGoRel.java]
                + [InstituteCountryRel.java][main/java/com/bio4j/neo4j/model/relationships/InstituteCountryRel.java]
                + [IsoformEventGeneratorRel.java][main/java/com/bio4j/neo4j/model/relationships/IsoformEventGeneratorRel.java]
                + ncbi
                  + [NCBIMainTaxonRel.java][main/java/com/bio4j/neo4j/model/relationships/ncbi/NCBIMainTaxonRel.java]
                  + [NCBITaxonParentRel.java][main/java/com/bio4j/neo4j/model/relationships/ncbi/NCBITaxonParentRel.java]
                  + [NCBITaxonRel.java][main/java/com/bio4j/neo4j/model/relationships/ncbi/NCBITaxonRel.java]
                + protein
                  + [BasicProteinSequenceCautionRel.java][main/java/com/bio4j/neo4j/model/relationships/protein/BasicProteinSequenceCautionRel.java]
                  + [ProteinDatasetRel.java][main/java/com/bio4j/neo4j/model/relationships/protein/ProteinDatasetRel.java]
                  + [ProteinEnzymaticActivityRel.java][main/java/com/bio4j/neo4j/model/relationships/protein/ProteinEnzymaticActivityRel.java]
                  + [ProteinErroneousGeneModelPredictionRel.java][main/java/com/bio4j/neo4j/model/relationships/protein/ProteinErroneousGeneModelPredictionRel.java]
                  + [ProteinErroneousInitiationRel.java][main/java/com/bio4j/neo4j/model/relationships/protein/ProteinErroneousInitiationRel.java]
                  + [ProteinErroneousTerminationRel.java][main/java/com/bio4j/neo4j/model/relationships/protein/ProteinErroneousTerminationRel.java]
                  + [ProteinErroneousTranslationRel.java][main/java/com/bio4j/neo4j/model/relationships/protein/ProteinErroneousTranslationRel.java]
                  + [ProteinFrameshiftRel.java][main/java/com/bio4j/neo4j/model/relationships/protein/ProteinFrameshiftRel.java]
                  + [ProteinGenomeElementRel.java][main/java/com/bio4j/neo4j/model/relationships/protein/ProteinGenomeElementRel.java]
                  + [ProteinGoRel.java][main/java/com/bio4j/neo4j/model/relationships/protein/ProteinGoRel.java]
                  + [ProteinInterproRel.java][main/java/com/bio4j/neo4j/model/relationships/protein/ProteinInterproRel.java]
                  + [ProteinIsoformInteractionRel.java][main/java/com/bio4j/neo4j/model/relationships/protein/ProteinIsoformInteractionRel.java]
                  + [ProteinIsoformRel.java][main/java/com/bio4j/neo4j/model/relationships/protein/ProteinIsoformRel.java]
                  + [ProteinKeywordRel.java][main/java/com/bio4j/neo4j/model/relationships/protein/ProteinKeywordRel.java]
                  + [ProteinMiscellaneousDiscrepancyRel.java][main/java/com/bio4j/neo4j/model/relationships/protein/ProteinMiscellaneousDiscrepancyRel.java]
                  + [ProteinOrganismRel.java][main/java/com/bio4j/neo4j/model/relationships/protein/ProteinOrganismRel.java]
                  + [ProteinPfamRel.java][main/java/com/bio4j/neo4j/model/relationships/protein/ProteinPfamRel.java]
                  + [ProteinProteinInteractionRel.java][main/java/com/bio4j/neo4j/model/relationships/protein/ProteinProteinInteractionRel.java]
                  + [ProteinReactomeRel.java][main/java/com/bio4j/neo4j/model/relationships/protein/ProteinReactomeRel.java]
                  + [ProteinSubcellularLocationRel.java][main/java/com/bio4j/neo4j/model/relationships/protein/ProteinSubcellularLocationRel.java]
                + refseq
                  + [GenomeElementCDSRel.java][main/java/com/bio4j/neo4j/model/relationships/refseq/GenomeElementCDSRel.java]
                  + [GenomeElementGeneRel.java][main/java/com/bio4j/neo4j/model/relationships/refseq/GenomeElementGeneRel.java]
                  + [GenomeElementMiscRnaRel.java][main/java/com/bio4j/neo4j/model/relationships/refseq/GenomeElementMiscRnaRel.java]
                  + [GenomeElementMRnaRel.java][main/java/com/bio4j/neo4j/model/relationships/refseq/GenomeElementMRnaRel.java]
                  + [GenomeElementNcRnaRel.java][main/java/com/bio4j/neo4j/model/relationships/refseq/GenomeElementNcRnaRel.java]
                  + [GenomeElementRRnaRel.java][main/java/com/bio4j/neo4j/model/relationships/refseq/GenomeElementRRnaRel.java]
                  + [GenomeElementTmRnaRel.java][main/java/com/bio4j/neo4j/model/relationships/refseq/GenomeElementTmRnaRel.java]
                  + [GenomeElementTRnaRel.java][main/java/com/bio4j/neo4j/model/relationships/refseq/GenomeElementTRnaRel.java]
                + sc
                  + [ErroneousGeneModelPredictionRel.java][main/java/com/bio4j/neo4j/model/relationships/sc/ErroneousGeneModelPredictionRel.java]
                  + [ErroneousInitiationRel.java][main/java/com/bio4j/neo4j/model/relationships/sc/ErroneousInitiationRel.java]
                  + [ErroneousTerminationRel.java][main/java/com/bio4j/neo4j/model/relationships/sc/ErroneousTerminationRel.java]
                  + [ErroneousTranslationRel.java][main/java/com/bio4j/neo4j/model/relationships/sc/ErroneousTranslationRel.java]
                  + [FrameshiftRel.java][main/java/com/bio4j/neo4j/model/relationships/sc/FrameshiftRel.java]
                  + [MiscellaneousDiscrepancyRel.java][main/java/com/bio4j/neo4j/model/relationships/sc/MiscellaneousDiscrepancyRel.java]
                + [SubcellularLocationParentRel.java][main/java/com/bio4j/neo4j/model/relationships/SubcellularLocationParentRel.java]
                + [TaxonParentRel.java][main/java/com/bio4j/neo4j/model/relationships/TaxonParentRel.java]
                + uniref
                  + [UniRef100MemberRel.java][main/java/com/bio4j/neo4j/model/relationships/uniref/UniRef100MemberRel.java]
                  + [UniRef50MemberRel.java][main/java/com/bio4j/neo4j/model/relationships/uniref/UniRef50MemberRel.java]
                  + [UniRef90MemberRel.java][main/java/com/bio4j/neo4j/model/relationships/uniref/UniRef90MemberRel.java]
              + util
                + [Bio4jManager.java][main/java/com/bio4j/neo4j/model/util/Bio4jManager.java]
                + [GoUtil.java][main/java/com/bio4j/neo4j/model/util/GoUtil.java]
                + [NodeIndexer.java][main/java/com/bio4j/neo4j/model/util/NodeIndexer.java]
                + [NodeRetriever.java][main/java/com/bio4j/neo4j/model/util/NodeRetriever.java]
                + [UniprotStuff.java][main/java/com/bio4j/neo4j/model/util/UniprotStuff.java]
            + [Neo4jManager.java][main/java/com/bio4j/neo4j/Neo4jManager.java]
            + programs
              + [GetProteinData.java][main/java/com/bio4j/neo4j/programs/GetProteinData.java]
              + [ImportEnzymeDB.java][main/java/com/bio4j/neo4j/programs/ImportEnzymeDB.java]
              + [ImportGeneOntology.java][main/java/com/bio4j/neo4j/programs/ImportGeneOntology.java]
              + [ImportIsoformSequences.java][main/java/com/bio4j/neo4j/programs/ImportIsoformSequences.java]
              + [ImportNCBITaxonomy.java][main/java/com/bio4j/neo4j/programs/ImportNCBITaxonomy.java]
              + [ImportNeo4jDB.java][main/java/com/bio4j/neo4j/programs/ImportNeo4jDB.java]
              + [ImportProteinInteractions.java][main/java/com/bio4j/neo4j/programs/ImportProteinInteractions.java]
              + [ImportRefSeq.java][main/java/com/bio4j/neo4j/programs/ImportRefSeq.java]
              + [ImportUniprot.java][main/java/com/bio4j/neo4j/programs/ImportUniprot.java]
              + [ImportUniref.java][main/java/com/bio4j/neo4j/programs/ImportUniref.java]
              + [IndexNCBITaxonomyByGiId.java][main/java/com/bio4j/neo4j/programs/IndexNCBITaxonomyByGiId.java]
              + [InitBio4jDB.java][main/java/com/bio4j/neo4j/programs/InitBio4jDB.java]
              + [UploadRefSeqSequencesToS3.java][main/java/com/bio4j/neo4j/programs/UploadRefSeqSequencesToS3.java]
        + ohnosequences
          + util
            + [Executable.java][main/java/com/ohnosequences/util/Executable.java]
            + [ExecuteFromFile.java][main/java/com/ohnosequences/util/ExecuteFromFile.java]
            + genbank
              + [GBCommon.java][main/java/com/ohnosequences/util/genbank/GBCommon.java]
          + xml
            + api
              + interfaces
                + [IAttribute.java][main/java/com/ohnosequences/xml/api/interfaces/IAttribute.java]
                + [IElement.java][main/java/com/ohnosequences/xml/api/interfaces/IElement.java]
                + [INameSpace.java][main/java/com/ohnosequences/xml/api/interfaces/INameSpace.java]
                + [IXmlThing.java][main/java/com/ohnosequences/xml/api/interfaces/IXmlThing.java]
                + [package-info.java][main/java/com/ohnosequences/xml/api/interfaces/package-info.java]
              + model
                + [NameSpace.java][main/java/com/ohnosequences/xml/api/model/NameSpace.java]
                + [package-info.java][main/java/com/ohnosequences/xml/api/model/package-info.java]
                + [XMLAttribute.java][main/java/com/ohnosequences/xml/api/model/XMLAttribute.java]
                + [XMLElement.java][main/java/com/ohnosequences/xml/api/model/XMLElement.java]
                + [XMLElementException.java][main/java/com/ohnosequences/xml/api/model/XMLElementException.java]
              + util
                + [XMLUtil.java][main/java/com/ohnosequences/xml/api/util/XMLUtil.java]
            + model
              + bio4j
                + [Bio4jNodeIndexXML.java][main/java/com/ohnosequences/xml/model/bio4j/Bio4jNodeIndexXML.java]
                + [Bio4jNodeXML.java][main/java/com/ohnosequences/xml/model/bio4j/Bio4jNodeXML.java]
                + [Bio4jPropertyXML.java][main/java/com/ohnosequences/xml/model/bio4j/Bio4jPropertyXML.java]
                + [Bio4jRelationshipIndexXML.java][main/java/com/ohnosequences/xml/model/bio4j/Bio4jRelationshipIndexXML.java]
                + [Bio4jRelationshipXML.java][main/java/com/ohnosequences/xml/model/bio4j/Bio4jRelationshipXML.java]
                + [UniprotDataXML.java][main/java/com/ohnosequences/xml/model/bio4j/UniprotDataXML.java]
              + go
                + [GoAnnotationXML.java][main/java/com/ohnosequences/xml/model/go/GoAnnotationXML.java]
                + [GOSlimXML.java][main/java/com/ohnosequences/xml/model/go/GOSlimXML.java]
                + [GoTermXML.java][main/java/com/ohnosequences/xml/model/go/GoTermXML.java]
                + [SlimSetXML.java][main/java/com/ohnosequences/xml/model/go/SlimSetXML.java]
              + uniprot
                + [ArticleXML.java][main/java/com/ohnosequences/xml/model/uniprot/ArticleXML.java]
                + [CommentXML.java][main/java/com/ohnosequences/xml/model/uniprot/CommentXML.java]
                + [FeatureXML.java][main/java/com/ohnosequences/xml/model/uniprot/FeatureXML.java]
                + [InterproXML.java][main/java/com/ohnosequences/xml/model/uniprot/InterproXML.java]
                + [IsoformXML.java][main/java/com/ohnosequences/xml/model/uniprot/IsoformXML.java]
                + [KeywordXML.java][main/java/com/ohnosequences/xml/model/uniprot/KeywordXML.java]
                + [ProteinXML.java][main/java/com/ohnosequences/xml/model/uniprot/ProteinXML.java]
                + [SubcellularLocationXML.java][main/java/com/ohnosequences/xml/model/uniprot/SubcellularLocationXML.java]
              + util
                + [Argument.java][main/java/com/ohnosequences/xml/model/util/Argument.java]
                + [Arguments.java][main/java/com/ohnosequences/xml/model/util/Arguments.java]
                + [Error.java][main/java/com/ohnosequences/xml/model/util/Error.java]
                + [Execution.java][main/java/com/ohnosequences/xml/model/util/Execution.java]
                + [FlexXMLWrapperClassCreator.java][main/java/com/ohnosequences/xml/model/util/FlexXMLWrapperClassCreator.java]
                + [ScheduledExecutions.java][main/java/com/ohnosequences/xml/model/util/ScheduledExecutions.java]
                + [XMLWrapperClass.java][main/java/com/ohnosequences/xml/model/util/XMLWrapperClass.java]
                + [XMLWrapperClassCreator.java][main/java/com/ohnosequences/xml/model/util/XMLWrapperClassCreator.java]

[main/java/com/bio4j/neo4j/BasicEntity.java]: ../BasicEntity.java.md
[main/java/com/bio4j/neo4j/BasicRelationship.java]: ../BasicRelationship.java.md
[main/java/com/bio4j/neo4j/codesamples/BiodieselProductionSample.java]: ../codesamples/BiodieselProductionSample.java.md
[main/java/com/bio4j/neo4j/codesamples/GetEnzymeData.java]: ../codesamples/GetEnzymeData.java.md
[main/java/com/bio4j/neo4j/codesamples/GetGenesInfo.java]: ../codesamples/GetGenesInfo.java.md
[main/java/com/bio4j/neo4j/codesamples/GetGOAnnotationsForOrganism.java]: ../codesamples/GetGOAnnotationsForOrganism.java.md
[main/java/com/bio4j/neo4j/codesamples/GetProteinsWithInterpro.java]: ../codesamples/GetProteinsWithInterpro.java.md
[main/java/com/bio4j/neo4j/codesamples/RealUseCase1.java]: ../codesamples/RealUseCase1.java.md
[main/java/com/bio4j/neo4j/codesamples/RetrieveProteinSample.java]: ../codesamples/RetrieveProteinSample.java.md
[main/java/com/bio4j/neo4j/model/nodes/AlternativeProductNode.java]: ../model/nodes/AlternativeProductNode.java.md
[main/java/com/bio4j/neo4j/model/nodes/citation/ArticleNode.java]: ../model/nodes/citation/ArticleNode.java.md
[main/java/com/bio4j/neo4j/model/nodes/citation/BookNode.java]: ../model/nodes/citation/BookNode.java.md
[main/java/com/bio4j/neo4j/model/nodes/citation/DBNode.java]: ../model/nodes/citation/DBNode.java.md
[main/java/com/bio4j/neo4j/model/nodes/citation/JournalNode.java]: ../model/nodes/citation/JournalNode.java.md
[main/java/com/bio4j/neo4j/model/nodes/citation/OnlineArticleNode.java]: ../model/nodes/citation/OnlineArticleNode.java.md
[main/java/com/bio4j/neo4j/model/nodes/citation/OnlineJournalNode.java]: ../model/nodes/citation/OnlineJournalNode.java.md
[main/java/com/bio4j/neo4j/model/nodes/citation/PatentNode.java]: ../model/nodes/citation/PatentNode.java.md
[main/java/com/bio4j/neo4j/model/nodes/citation/PublisherNode.java]: ../model/nodes/citation/PublisherNode.java.md
[main/java/com/bio4j/neo4j/model/nodes/citation/SubmissionNode.java]: ../model/nodes/citation/SubmissionNode.java.md
[main/java/com/bio4j/neo4j/model/nodes/citation/ThesisNode.java]: ../model/nodes/citation/ThesisNode.java.md
[main/java/com/bio4j/neo4j/model/nodes/citation/UnpublishedObservationNode.java]: ../model/nodes/citation/UnpublishedObservationNode.java.md
[main/java/com/bio4j/neo4j/model/nodes/CityNode.java]: ../model/nodes/CityNode.java.md
[main/java/com/bio4j/neo4j/model/nodes/CommentTypeNode.java]: ../model/nodes/CommentTypeNode.java.md
[main/java/com/bio4j/neo4j/model/nodes/ConsortiumNode.java]: ../model/nodes/ConsortiumNode.java.md
[main/java/com/bio4j/neo4j/model/nodes/CountryNode.java]: ../model/nodes/CountryNode.java.md
[main/java/com/bio4j/neo4j/model/nodes/DatasetNode.java]: ../model/nodes/DatasetNode.java.md
[main/java/com/bio4j/neo4j/model/nodes/EnzymeNode.java]: ../model/nodes/EnzymeNode.java.md
[main/java/com/bio4j/neo4j/model/nodes/FeatureTypeNode.java]: ../model/nodes/FeatureTypeNode.java.md
[main/java/com/bio4j/neo4j/model/nodes/GoTermNode.java]: ../model/nodes/GoTermNode.java.md
[main/java/com/bio4j/neo4j/model/nodes/InstituteNode.java]: ../model/nodes/InstituteNode.java.md
[main/java/com/bio4j/neo4j/model/nodes/InterproNode.java]: ../model/nodes/InterproNode.java.md
[main/java/com/bio4j/neo4j/model/nodes/IsoformNode.java]: ../model/nodes/IsoformNode.java.md
[main/java/com/bio4j/neo4j/model/nodes/KeywordNode.java]: ../model/nodes/KeywordNode.java.md
[main/java/com/bio4j/neo4j/model/nodes/ncbi/NCBITaxonNode.java]: ../model/nodes/ncbi/NCBITaxonNode.java.md
[main/java/com/bio4j/neo4j/model/nodes/OrganismNode.java]: ../model/nodes/OrganismNode.java.md
[main/java/com/bio4j/neo4j/model/nodes/PersonNode.java]: ../model/nodes/PersonNode.java.md
[main/java/com/bio4j/neo4j/model/nodes/PfamNode.java]: ../model/nodes/PfamNode.java.md
[main/java/com/bio4j/neo4j/model/nodes/ProteinNode.java]: ../model/nodes/ProteinNode.java.md
[main/java/com/bio4j/neo4j/model/nodes/reactome/ReactomeTermNode.java]: ../model/nodes/reactome/ReactomeTermNode.java.md
[main/java/com/bio4j/neo4j/model/nodes/refseq/CDSNode.java]: ../model/nodes/refseq/CDSNode.java.md
[main/java/com/bio4j/neo4j/model/nodes/refseq/GeneNode.java]: ../model/nodes/refseq/GeneNode.java.md
[main/java/com/bio4j/neo4j/model/nodes/refseq/GenomeElementNode.java]: ../model/nodes/refseq/GenomeElementNode.java.md
[main/java/com/bio4j/neo4j/model/nodes/refseq/rna/MiscRNANode.java]: ../model/nodes/refseq/rna/MiscRNANode.java.md
[main/java/com/bio4j/neo4j/model/nodes/refseq/rna/MRNANode.java]: ../model/nodes/refseq/rna/MRNANode.java.md
[main/java/com/bio4j/neo4j/model/nodes/refseq/rna/NcRNANode.java]: ../model/nodes/refseq/rna/NcRNANode.java.md
[main/java/com/bio4j/neo4j/model/nodes/refseq/rna/RNANode.java]: ../model/nodes/refseq/rna/RNANode.java.md
[main/java/com/bio4j/neo4j/model/nodes/refseq/rna/RRNANode.java]: ../model/nodes/refseq/rna/RRNANode.java.md
[main/java/com/bio4j/neo4j/model/nodes/refseq/rna/TmRNANode.java]: ../model/nodes/refseq/rna/TmRNANode.java.md
[main/java/com/bio4j/neo4j/model/nodes/refseq/rna/TRNANode.java]: ../model/nodes/refseq/rna/TRNANode.java.md
[main/java/com/bio4j/neo4j/model/nodes/SequenceCautionNode.java]: ../model/nodes/SequenceCautionNode.java.md
[main/java/com/bio4j/neo4j/model/nodes/SubcellularLocationNode.java]: ../model/nodes/SubcellularLocationNode.java.md
[main/java/com/bio4j/neo4j/model/nodes/TaxonNode.java]: ../model/nodes/TaxonNode.java.md
[main/java/com/bio4j/neo4j/model/relationships/aproducts/AlternativeProductInitiationRel.java]: ../model/relationships/aproducts/AlternativeProductInitiationRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/aproducts/AlternativeProductPromoterRel.java]: ../model/relationships/aproducts/AlternativeProductPromoterRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/aproducts/AlternativeProductRibosomalFrameshiftingRel.java]: ../model/relationships/aproducts/AlternativeProductRibosomalFrameshiftingRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/aproducts/AlternativeProductSplicingRel.java]: ../model/relationships/aproducts/AlternativeProductSplicingRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/citation/article/ArticleAuthorRel.java]: ../model/relationships/citation/article/ArticleAuthorRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/citation/article/ArticleJournalRel.java]: ../model/relationships/citation/article/ArticleJournalRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/citation/article/ArticleProteinCitationRel.java]: ../model/relationships/citation/article/ArticleProteinCitationRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/citation/book/BookAuthorRel.java]: ../model/relationships/citation/book/BookAuthorRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/citation/book/BookCityRel.java]: ../model/relationships/citation/book/BookCityRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/citation/book/BookEditorRel.java]: ../model/relationships/citation/book/BookEditorRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/citation/book/BookProteinCitationRel.java]: ../model/relationships/citation/book/BookProteinCitationRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/citation/book/BookPublisherRel.java]: ../model/relationships/citation/book/BookPublisherRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/citation/onarticle/OnlineArticleAuthorRel.java]: ../model/relationships/citation/onarticle/OnlineArticleAuthorRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/citation/onarticle/OnlineArticleJournalRel.java]: ../model/relationships/citation/onarticle/OnlineArticleJournalRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/citation/onarticle/OnlineArticleProteinCitationRel.java]: ../model/relationships/citation/onarticle/OnlineArticleProteinCitationRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/citation/patent/PatentAuthorRel.java]: ../model/relationships/citation/patent/PatentAuthorRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/citation/patent/PatentProteinCitationRel.java]: ../model/relationships/citation/patent/PatentProteinCitationRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/citation/submission/SubmissionAuthorRel.java]: ../model/relationships/citation/submission/SubmissionAuthorRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/citation/submission/SubmissionDbRel.java]: ../model/relationships/citation/submission/SubmissionDbRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/citation/submission/SubmissionProteinCitationRel.java]: ../model/relationships/citation/submission/SubmissionProteinCitationRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/citation/thesis/ThesisAuthorRel.java]: ../model/relationships/citation/thesis/ThesisAuthorRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/citation/thesis/ThesisInstituteRel.java]: ../model/relationships/citation/thesis/ThesisInstituteRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/citation/thesis/ThesisProteinCitationRel.java]: ../model/relationships/citation/thesis/ThesisProteinCitationRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/citation/uo/UnpublishedObservationAuthorRel.java]: ../model/relationships/citation/uo/UnpublishedObservationAuthorRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/citation/uo/UnpublishedObservationProteinCitationRel.java]: ../model/relationships/citation/uo/UnpublishedObservationProteinCitationRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/comment/AllergenCommentRel.java]: ../model/relationships/comment/AllergenCommentRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/comment/BasicCommentRel.java]: ../model/relationships/comment/BasicCommentRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/comment/BioPhysicoChemicalPropertiesCommentRel.java]: ../model/relationships/comment/BioPhysicoChemicalPropertiesCommentRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/comment/BiotechnologyCommentRel.java]: ../model/relationships/comment/BiotechnologyCommentRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/comment/CatalyticActivityCommentRel.java]: ../model/relationships/comment/CatalyticActivityCommentRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/comment/CautionCommentRel.java]: ../model/relationships/comment/CautionCommentRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/comment/CofactorCommentRel.java]: ../model/relationships/comment/CofactorCommentRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/comment/DevelopmentalStageCommentRel.java]: ../model/relationships/comment/DevelopmentalStageCommentRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/comment/DiseaseCommentRel.java]: ../model/relationships/comment/DiseaseCommentRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/comment/DisruptionPhenotypeCommentRel.java]: ../model/relationships/comment/DisruptionPhenotypeCommentRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/comment/DomainCommentRel.java]: ../model/relationships/comment/DomainCommentRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/comment/EnzymeRegulationCommentRel.java]: ../model/relationships/comment/EnzymeRegulationCommentRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/comment/FunctionCommentRel.java]: ../model/relationships/comment/FunctionCommentRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/comment/InductionCommentRel.java]: ../model/relationships/comment/InductionCommentRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/comment/MassSpectrometryCommentRel.java]: ../model/relationships/comment/MassSpectrometryCommentRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/comment/MiscellaneousCommentRel.java]: ../model/relationships/comment/MiscellaneousCommentRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/comment/OnlineInformationCommentRel.java]: ../model/relationships/comment/OnlineInformationCommentRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/comment/PathwayCommentRel.java]: ../model/relationships/comment/PathwayCommentRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/comment/PharmaceuticalCommentRel.java]: ../model/relationships/comment/PharmaceuticalCommentRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/comment/PolymorphismCommentRel.java]: ../model/relationships/comment/PolymorphismCommentRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/comment/PostTranslationalModificationCommentRel.java]: ../model/relationships/comment/PostTranslationalModificationCommentRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/comment/RnaEditingCommentRel.java]: ../model/relationships/comment/RnaEditingCommentRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/comment/SimilarityCommentRel.java]: ../model/relationships/comment/SimilarityCommentRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/comment/SubunitCommentRel.java]: ../model/relationships/comment/SubunitCommentRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/comment/TissueSpecificityCommentRel.java]: ../model/relationships/comment/TissueSpecificityCommentRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/comment/ToxicDoseCommentRel.java]: ../model/relationships/comment/ToxicDoseCommentRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/features/ActiveSiteFeatureRel.java]: ../model/relationships/features/ActiveSiteFeatureRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/features/BasicFeatureRel.java]: ../model/relationships/features/BasicFeatureRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/features/BindingSiteFeatureRel.java]: ../model/relationships/features/BindingSiteFeatureRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/features/CalciumBindingRegionFeatureRel.java]: ../model/relationships/features/CalciumBindingRegionFeatureRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/features/ChainFeatureRel.java]: ../model/relationships/features/ChainFeatureRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/features/CoiledCoilRegionFeatureRel.java]: ../model/relationships/features/CoiledCoilRegionFeatureRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/features/CompositionallyBiasedRegionFeatureRel.java]: ../model/relationships/features/CompositionallyBiasedRegionFeatureRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/features/CrossLinkFeatureRel.java]: ../model/relationships/features/CrossLinkFeatureRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/features/DisulfideBondFeatureRel.java]: ../model/relationships/features/DisulfideBondFeatureRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/features/DnaBindingRegionFeatureRel.java]: ../model/relationships/features/DnaBindingRegionFeatureRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/features/DomainFeatureRel.java]: ../model/relationships/features/DomainFeatureRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/features/GlycosylationSiteFeatureRel.java]: ../model/relationships/features/GlycosylationSiteFeatureRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/features/HelixFeatureRel.java]: ../model/relationships/features/HelixFeatureRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/features/InitiatorMethionineFeatureRel.java]: ../model/relationships/features/InitiatorMethionineFeatureRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/features/IntramembraneRegionFeatureRel.java]: ../model/relationships/features/IntramembraneRegionFeatureRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/features/LipidMoietyBindingRegionFeatureRel.java]: ../model/relationships/features/LipidMoietyBindingRegionFeatureRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/features/MetalIonBindingSiteFeatureRel.java]: ../model/relationships/features/MetalIonBindingSiteFeatureRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/features/ModifiedResidueFeatureRel.java]: ../model/relationships/features/ModifiedResidueFeatureRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/features/MutagenesisSiteFeatureRel.java]: ../model/relationships/features/MutagenesisSiteFeatureRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/features/NonConsecutiveResiduesFeatureRel.java]: ../model/relationships/features/NonConsecutiveResiduesFeatureRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/features/NonStandardAminoAcidFeatureRel.java]: ../model/relationships/features/NonStandardAminoAcidFeatureRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/features/NonTerminalResidueFeatureRel.java]: ../model/relationships/features/NonTerminalResidueFeatureRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/features/NucleotidePhosphateBindingRegionFeatureRel.java]: ../model/relationships/features/NucleotidePhosphateBindingRegionFeatureRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/features/PeptideFeatureRel.java]: ../model/relationships/features/PeptideFeatureRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/features/PropeptideFeatureRel.java]: ../model/relationships/features/PropeptideFeatureRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/features/RegionOfInterestFeatureRel.java]: ../model/relationships/features/RegionOfInterestFeatureRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/features/RepeatFeatureRel.java]: ../model/relationships/features/RepeatFeatureRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/features/SequenceConflictFeatureRel.java]: ../model/relationships/features/SequenceConflictFeatureRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/features/SequenceVariantFeatureRel.java]: ../model/relationships/features/SequenceVariantFeatureRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/features/ShortSequenceMotifFeatureRel.java]: ../model/relationships/features/ShortSequenceMotifFeatureRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/features/SignalPeptideFeatureRel.java]: ../model/relationships/features/SignalPeptideFeatureRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/features/SiteFeatureRel.java]: ../model/relationships/features/SiteFeatureRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/features/SpliceVariantFeatureRel.java]: ../model/relationships/features/SpliceVariantFeatureRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/features/StrandFeatureRel.java]: ../model/relationships/features/StrandFeatureRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/features/TopologicalDomainFeatureRel.java]: ../model/relationships/features/TopologicalDomainFeatureRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/features/TransitPeptideFeatureRel.java]: ../model/relationships/features/TransitPeptideFeatureRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/features/TransmembraneRegionFeatureRel.java]: ../model/relationships/features/TransmembraneRegionFeatureRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/features/TurnFeatureRel.java]: ../model/relationships/features/TurnFeatureRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/features/UnsureResidueFeatureRel.java]: ../model/relationships/features/UnsureResidueFeatureRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/features/ZincFingerRegionFeatureRel.java]: ../model/relationships/features/ZincFingerRegionFeatureRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/go/BiologicalProcessRel.java]: ../model/relationships/go/BiologicalProcessRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/go/CellularComponentRel.java]: ../model/relationships/go/CellularComponentRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/go/HasPartOfGoRel.java]: ../model/relationships/go/HasPartOfGoRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/go/IsAGoRel.java]: ../model/relationships/go/IsAGoRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/go/MainGoRel.java]: ../model/relationships/go/MainGoRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/go/MolecularFunctionRel.java]: ../model/relationships/go/MolecularFunctionRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/go/NegativelyRegulatesGoRel.java]: ../model/relationships/go/NegativelyRegulatesGoRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/go/PartOfGoRel.java]: ../model/relationships/go/PartOfGoRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/go/PositivelyRegulatesGoRel.java]: ../model/relationships/go/PositivelyRegulatesGoRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/go/RegulatesGoRel.java]: ../model/relationships/go/RegulatesGoRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/InstituteCountryRel.java]: ../model/relationships/InstituteCountryRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/IsoformEventGeneratorRel.java]: ../model/relationships/IsoformEventGeneratorRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/ncbi/NCBIMainTaxonRel.java]: ../model/relationships/ncbi/NCBIMainTaxonRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/ncbi/NCBITaxonParentRel.java]: ../model/relationships/ncbi/NCBITaxonParentRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/ncbi/NCBITaxonRel.java]: ../model/relationships/ncbi/NCBITaxonRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/protein/BasicProteinSequenceCautionRel.java]: ../model/relationships/protein/BasicProteinSequenceCautionRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/protein/ProteinDatasetRel.java]: ../model/relationships/protein/ProteinDatasetRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/protein/ProteinEnzymaticActivityRel.java]: ../model/relationships/protein/ProteinEnzymaticActivityRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/protein/ProteinErroneousGeneModelPredictionRel.java]: ../model/relationships/protein/ProteinErroneousGeneModelPredictionRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/protein/ProteinErroneousInitiationRel.java]: ../model/relationships/protein/ProteinErroneousInitiationRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/protein/ProteinErroneousTerminationRel.java]: ../model/relationships/protein/ProteinErroneousTerminationRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/protein/ProteinErroneousTranslationRel.java]: ../model/relationships/protein/ProteinErroneousTranslationRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/protein/ProteinFrameshiftRel.java]: ../model/relationships/protein/ProteinFrameshiftRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/protein/ProteinGenomeElementRel.java]: ../model/relationships/protein/ProteinGenomeElementRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/protein/ProteinGoRel.java]: ../model/relationships/protein/ProteinGoRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/protein/ProteinInterproRel.java]: ../model/relationships/protein/ProteinInterproRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/protein/ProteinIsoformInteractionRel.java]: ../model/relationships/protein/ProteinIsoformInteractionRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/protein/ProteinIsoformRel.java]: ../model/relationships/protein/ProteinIsoformRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/protein/ProteinKeywordRel.java]: ../model/relationships/protein/ProteinKeywordRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/protein/ProteinMiscellaneousDiscrepancyRel.java]: ../model/relationships/protein/ProteinMiscellaneousDiscrepancyRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/protein/ProteinOrganismRel.java]: ../model/relationships/protein/ProteinOrganismRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/protein/ProteinPfamRel.java]: ../model/relationships/protein/ProteinPfamRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/protein/ProteinProteinInteractionRel.java]: ../model/relationships/protein/ProteinProteinInteractionRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/protein/ProteinReactomeRel.java]: ../model/relationships/protein/ProteinReactomeRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/protein/ProteinSubcellularLocationRel.java]: ../model/relationships/protein/ProteinSubcellularLocationRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/refseq/GenomeElementCDSRel.java]: ../model/relationships/refseq/GenomeElementCDSRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/refseq/GenomeElementGeneRel.java]: ../model/relationships/refseq/GenomeElementGeneRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/refseq/GenomeElementMiscRnaRel.java]: ../model/relationships/refseq/GenomeElementMiscRnaRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/refseq/GenomeElementMRnaRel.java]: ../model/relationships/refseq/GenomeElementMRnaRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/refseq/GenomeElementNcRnaRel.java]: ../model/relationships/refseq/GenomeElementNcRnaRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/refseq/GenomeElementRRnaRel.java]: ../model/relationships/refseq/GenomeElementRRnaRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/refseq/GenomeElementTmRnaRel.java]: ../model/relationships/refseq/GenomeElementTmRnaRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/refseq/GenomeElementTRnaRel.java]: ../model/relationships/refseq/GenomeElementTRnaRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/sc/ErroneousGeneModelPredictionRel.java]: ../model/relationships/sc/ErroneousGeneModelPredictionRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/sc/ErroneousInitiationRel.java]: ../model/relationships/sc/ErroneousInitiationRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/sc/ErroneousTerminationRel.java]: ../model/relationships/sc/ErroneousTerminationRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/sc/ErroneousTranslationRel.java]: ../model/relationships/sc/ErroneousTranslationRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/sc/FrameshiftRel.java]: ../model/relationships/sc/FrameshiftRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/sc/MiscellaneousDiscrepancyRel.java]: ../model/relationships/sc/MiscellaneousDiscrepancyRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/SubcellularLocationParentRel.java]: ../model/relationships/SubcellularLocationParentRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/TaxonParentRel.java]: ../model/relationships/TaxonParentRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/uniref/UniRef100MemberRel.java]: ../model/relationships/uniref/UniRef100MemberRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/uniref/UniRef50MemberRel.java]: ../model/relationships/uniref/UniRef50MemberRel.java.md
[main/java/com/bio4j/neo4j/model/relationships/uniref/UniRef90MemberRel.java]: ../model/relationships/uniref/UniRef90MemberRel.java.md
[main/java/com/bio4j/neo4j/model/util/Bio4jManager.java]: ../model/util/Bio4jManager.java.md
[main/java/com/bio4j/neo4j/model/util/GoUtil.java]: ../model/util/GoUtil.java.md
[main/java/com/bio4j/neo4j/model/util/NodeIndexer.java]: ../model/util/NodeIndexer.java.md
[main/java/com/bio4j/neo4j/model/util/NodeRetriever.java]: ../model/util/NodeRetriever.java.md
[main/java/com/bio4j/neo4j/model/util/UniprotStuff.java]: ../model/util/UniprotStuff.java.md
[main/java/com/bio4j/neo4j/Neo4jManager.java]: ../Neo4jManager.java.md
[main/java/com/bio4j/neo4j/programs/GetProteinData.java]: GetProteinData.java.md
[main/java/com/bio4j/neo4j/programs/ImportEnzymeDB.java]: ImportEnzymeDB.java.md
[main/java/com/bio4j/neo4j/programs/ImportGeneOntology.java]: ImportGeneOntology.java.md
[main/java/com/bio4j/neo4j/programs/ImportIsoformSequences.java]: ImportIsoformSequences.java.md
[main/java/com/bio4j/neo4j/programs/ImportNCBITaxonomy.java]: ImportNCBITaxonomy.java.md
[main/java/com/bio4j/neo4j/programs/ImportNeo4jDB.java]: ImportNeo4jDB.java.md
[main/java/com/bio4j/neo4j/programs/ImportProteinInteractions.java]: ImportProteinInteractions.java.md
[main/java/com/bio4j/neo4j/programs/ImportRefSeq.java]: ImportRefSeq.java.md
[main/java/com/bio4j/neo4j/programs/ImportUniprot.java]: ImportUniprot.java.md
[main/java/com/bio4j/neo4j/programs/ImportUniref.java]: ImportUniref.java.md
[main/java/com/bio4j/neo4j/programs/IndexNCBITaxonomyByGiId.java]: IndexNCBITaxonomyByGiId.java.md
[main/java/com/bio4j/neo4j/programs/InitBio4jDB.java]: InitBio4jDB.java.md
[main/java/com/bio4j/neo4j/programs/UploadRefSeqSequencesToS3.java]: UploadRefSeqSequencesToS3.java.md
[main/java/com/ohnosequences/util/Executable.java]: ../../../ohnosequences/util/Executable.java.md
[main/java/com/ohnosequences/util/ExecuteFromFile.java]: ../../../ohnosequences/util/ExecuteFromFile.java.md
[main/java/com/ohnosequences/util/genbank/GBCommon.java]: ../../../ohnosequences/util/genbank/GBCommon.java.md
[main/java/com/ohnosequences/xml/api/interfaces/IAttribute.java]: ../../../ohnosequences/xml/api/interfaces/IAttribute.java.md
[main/java/com/ohnosequences/xml/api/interfaces/IElement.java]: ../../../ohnosequences/xml/api/interfaces/IElement.java.md
[main/java/com/ohnosequences/xml/api/interfaces/INameSpace.java]: ../../../ohnosequences/xml/api/interfaces/INameSpace.java.md
[main/java/com/ohnosequences/xml/api/interfaces/IXmlThing.java]: ../../../ohnosequences/xml/api/interfaces/IXmlThing.java.md
[main/java/com/ohnosequences/xml/api/interfaces/package-info.java]: ../../../ohnosequences/xml/api/interfaces/package-info.java.md
[main/java/com/ohnosequences/xml/api/model/NameSpace.java]: ../../../ohnosequences/xml/api/model/NameSpace.java.md
[main/java/com/ohnosequences/xml/api/model/package-info.java]: ../../../ohnosequences/xml/api/model/package-info.java.md
[main/java/com/ohnosequences/xml/api/model/XMLAttribute.java]: ../../../ohnosequences/xml/api/model/XMLAttribute.java.md
[main/java/com/ohnosequences/xml/api/model/XMLElement.java]: ../../../ohnosequences/xml/api/model/XMLElement.java.md
[main/java/com/ohnosequences/xml/api/model/XMLElementException.java]: ../../../ohnosequences/xml/api/model/XMLElementException.java.md
[main/java/com/ohnosequences/xml/api/util/XMLUtil.java]: ../../../ohnosequences/xml/api/util/XMLUtil.java.md
[main/java/com/ohnosequences/xml/model/bio4j/Bio4jNodeIndexXML.java]: ../../../ohnosequences/xml/model/bio4j/Bio4jNodeIndexXML.java.md
[main/java/com/ohnosequences/xml/model/bio4j/Bio4jNodeXML.java]: ../../../ohnosequences/xml/model/bio4j/Bio4jNodeXML.java.md
[main/java/com/ohnosequences/xml/model/bio4j/Bio4jPropertyXML.java]: ../../../ohnosequences/xml/model/bio4j/Bio4jPropertyXML.java.md
[main/java/com/ohnosequences/xml/model/bio4j/Bio4jRelationshipIndexXML.java]: ../../../ohnosequences/xml/model/bio4j/Bio4jRelationshipIndexXML.java.md
[main/java/com/ohnosequences/xml/model/bio4j/Bio4jRelationshipXML.java]: ../../../ohnosequences/xml/model/bio4j/Bio4jRelationshipXML.java.md
[main/java/com/ohnosequences/xml/model/bio4j/UniprotDataXML.java]: ../../../ohnosequences/xml/model/bio4j/UniprotDataXML.java.md
[main/java/com/ohnosequences/xml/model/go/GoAnnotationXML.java]: ../../../ohnosequences/xml/model/go/GoAnnotationXML.java.md
[main/java/com/ohnosequences/xml/model/go/GOSlimXML.java]: ../../../ohnosequences/xml/model/go/GOSlimXML.java.md
[main/java/com/ohnosequences/xml/model/go/GoTermXML.java]: ../../../ohnosequences/xml/model/go/GoTermXML.java.md
[main/java/com/ohnosequences/xml/model/go/SlimSetXML.java]: ../../../ohnosequences/xml/model/go/SlimSetXML.java.md
[main/java/com/ohnosequences/xml/model/uniprot/ArticleXML.java]: ../../../ohnosequences/xml/model/uniprot/ArticleXML.java.md
[main/java/com/ohnosequences/xml/model/uniprot/CommentXML.java]: ../../../ohnosequences/xml/model/uniprot/CommentXML.java.md
[main/java/com/ohnosequences/xml/model/uniprot/FeatureXML.java]: ../../../ohnosequences/xml/model/uniprot/FeatureXML.java.md
[main/java/com/ohnosequences/xml/model/uniprot/InterproXML.java]: ../../../ohnosequences/xml/model/uniprot/InterproXML.java.md
[main/java/com/ohnosequences/xml/model/uniprot/IsoformXML.java]: ../../../ohnosequences/xml/model/uniprot/IsoformXML.java.md
[main/java/com/ohnosequences/xml/model/uniprot/KeywordXML.java]: ../../../ohnosequences/xml/model/uniprot/KeywordXML.java.md
[main/java/com/ohnosequences/xml/model/uniprot/ProteinXML.java]: ../../../ohnosequences/xml/model/uniprot/ProteinXML.java.md
[main/java/com/ohnosequences/xml/model/uniprot/SubcellularLocationXML.java]: ../../../ohnosequences/xml/model/uniprot/SubcellularLocationXML.java.md
[main/java/com/ohnosequences/xml/model/util/Argument.java]: ../../../ohnosequences/xml/model/util/Argument.java.md
[main/java/com/ohnosequences/xml/model/util/Arguments.java]: ../../../ohnosequences/xml/model/util/Arguments.java.md
[main/java/com/ohnosequences/xml/model/util/Error.java]: ../../../ohnosequences/xml/model/util/Error.java.md
[main/java/com/ohnosequences/xml/model/util/Execution.java]: ../../../ohnosequences/xml/model/util/Execution.java.md
[main/java/com/ohnosequences/xml/model/util/FlexXMLWrapperClassCreator.java]: ../../../ohnosequences/xml/model/util/FlexXMLWrapperClassCreator.java.md
[main/java/com/ohnosequences/xml/model/util/ScheduledExecutions.java]: ../../../ohnosequences/xml/model/util/ScheduledExecutions.java.md
[main/java/com/ohnosequences/xml/model/util/XMLWrapperClass.java]: ../../../ohnosequences/xml/model/util/XMLWrapperClass.java.md
[main/java/com/ohnosequences/xml/model/util/XMLWrapperClassCreator.java]: ../../../ohnosequences/xml/model/util/XMLWrapperClassCreator.java.md