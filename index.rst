:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

Current Status
==============

Now that crosstalk removal has been descoped from the Camera DAQ, there is only one version of the pixel image that can be and needs to be retrieved, rather than two.
There are multiple destinations for this image:

* Camera Diagnostic Cluster for automated visualization and rapid analysis
* Active Optics System (AOS) for wavefront sensors only
* Observatory Operations Data Service (OODS) for automated and ad hoc (including human-driven) rapid analysis
* Data Backbone (DBB) for long-term reliable archival
* For most science and possibly calibration images, Prompt Processing for execution of automated processing pipelines (primarily the Alert Production).

The first four of these are located in Chile at either the Summit or Base.
The Prompt Processing systems are located at the LSST Data Facility (LDF), requiring transfer of the pixels over the international network.
The desired latency for all of these is generally "as rapid as possible", with the exception of the DBB, which has up to 24 hours.
The DBB also transfers data over the international network, but more slowly.

The first four of these are expecting to persist the pixel data as FITS files in RAM disk or persistent file storage (e.g. GPFS); as a result, it makes sense to do the same for Prompt Processing as well, though care should be taken to minimize latencies in order to avoid delaying alert generation.
All of these need to ingest the image files into a Data Butler repository to enable pipeline code to access the images using ``butler.get()``.
It is currently expected that these would be separate Butler repos.
The data ID used for the Butler should include either the image name or group ID and image ("snap") number as well as the raft and detector/CCD identifiers.
Each system needs to send an event to indicate that the image has been ingested and is thus available to pipelines.

The current baseline, as implemented for LATISS, has an image writer component of the Camera Control System (CCS) writing to the Camera Diagnostic Cluster.
Another instance of this image writer is intended to be configured and deployed for the AOS.
The OODS and DBB are fed by the DM Archiver and Forwarders, which are a separate image retrieval and writing system.
Prompt Processing has not yet been implemented, but it is supposed to use another instance of the Forwarders.
In addition, a Catch-Up Archiver is meant to feed the DBB with images that were otherwise missed, including those taken during a Summit-to-Base network outage.

An independent Header Service retrieves image metadata from events and telemetry, writing a single metadata object for each image that can be used by any image writer.

.. figure:: /_static/ImageWritingCurrent.png
    :name: Current Image Writing System

    The image writing system as currently baselined (somewhat simplified).
    The red arrows represent image-completion messaging from the Archiver to the OODS and "header" metadata transfer from the Header Service to the Forwarder (via the Large File Annex).


Simplifications
===============

Can some of these be combined?

Comparison of CCS with Archiver
-------------------------------

The CCS image writer has advantages compared to the Archiver/Forwarder:

* It is written by personnel who are physically and administratively close to the DAQ team, as opposed to NCSA.
* The application of which it is a part is the source of truth for many metadata and timing items as opposed to an independent CSC.
* It has the shortest latency requirements.
* It has been extensively tested on test stands at SLAC and Tucson, and more recently has undergone 9-raft focal-plane testing and ComCam testing.

But it also has disadvantages:

* It is written in Java using a JNI wrapping of the DAQ C++ API; Java FITS APIs do not have identical feature sets compared with the Stack's Python/C++ APIs.
  The Forwarder is written using the DAQ C++ API alone.
  (In both cases, workarounds for DAQ API misfeatures may have to be pushed upstream.)
* Its control code is currently written for "streaming" use, where the prime goal is to capture and analyze the most recent image taken (only), although future work is planned to retrieve images by name for visualization purposes.
  The Forwarder is written for "commanded" use, where the prime goal is to capture a specific image by name.
  The Catch-Up Archiver needs to iterate through the catalog of available images in the DAQ, retrieving specific images that have not already been archived.
* It will fail to capture data if a node fails during or between image captures; it must be manually reconfigured to recover to normal operation or else it continues to fail to capture data.
  The Forwarders are automatically tolerant to node failures between image captures and automatically recover (but lose data) if a node fails during image capture.
* It does not yet have Butler ingest or pipeline execution capability or cache management capability for a Butler repo (only a cron/find-based cleanup mechanism).
  The Archiver is already integrated with the OODS.
* It does not yet retrieve metadata written by the Header Service, although this is on the roadmap.

Catch-Up Archiver
-----------------

An independent Catch-Up Archiver will be needed in any case.
Neither the DM Archiver/Forwarder nor the CCS image writer can be considered 100% reliable in terms of capturing all science images.
The Catch-Up Archiver could potentially reuse code from the image writer or Forwarder for pixel manipulation and file output as well as transfer to the DBB.

The Catch-Up Archiver can potentially live at the Summit.
If 3 machines with 1 GB/sec (over 10Gb Ethernet) inbound and outbound network bandwidth are allocated to the Catch-Up Archiver, it should be possible to copy data to the Base at the rate of one 12 GB (uncompressed, even) image per 4 seconds, 4X the normal image capture rate, which is at most one image per 17 seconds.
This is sufficient to empty the buffer after even a long outage.
The Catch-Up Archiver does need to ingest to a Base-resident DBB and contact that DBB to know which images have already been archived.

Design Proposal
---------------

The primary simplification of the design allowed by the descoping of crosstalk images is the combining of the Prompt Processing Forwarder with the existing Archiver/Forwarder, as there is now only one image type.
But another major possible simplification is to remove the DM Archiver/Forwarder completely and use the CCS image writer in its place.
This can reduce the number of code bases that need to be supported, saving development effort.
It would remove two clients from the DAQ (Forwarder and Prompt Processing Forwarder).
It would remove the need for any DAQ clients to live at the Base, thereby removing the need for DAQ networking to extend over the DWDM[#1]_.
Since the CCS is absolutely necessary for image-taking, it would remove the possibility of images being taken with the Archiver or Prompt Processing disabled.
(Such images would eventually be retrieved by the Catch-Up Archiver, but they would be at least delayed for Prompt Processing purposes.)
It eliminates the current duplication of images and resultant confusion.
It ensures that engineering images taken under CCS control are captured the same way as images taken under OCS control (though possibly with less metadata).

.. [#1] DWDM stands for dense wavelength-division multiplexing, a way of using a single optical fiber to provide multiple independent Layer 2 network channels.
    DAQ networking requires a direct, unrouted Layer 2 connection between DAQ and client.
    These connections have been tested, but they are potentially more fragile due to the long distance and lower-level protocols involved.
    In this proposal, outbound traffic from the image writer is over Layer 3 TCP/IP network connections that can be routed.

.. figure:: /_static/ImageWritingProposed.png
    :name: Proposed Image Writing System

    The image writing system as proposed.
    The Archiver and Forwarder are removed.
    The Header Service metadata goes to the CCS Image Writer.
    Image-completion messages go from the CCS Image Writer to the OODS.
    Images are copied to the Handoff Filesystem.

The following additions would need to be made:

* Sufficient Summit-located compute resources, including hot spare nodes and network bandwidth, would need to be devoted to the Camera Diagnostic Cluster in order for it to also serve as the source of OODS, DBB, and Prompt Processing data.
  An alternative would be to use the same client codebase but run two instances, one on the Diagnostic Cluster as currently planned and one somewhere else for writing images to be be archived.
  The decision here is akin to the decision between RAID 5 (error correction) and RAID 1 (mirroring) for storage; the former is more efficient in terms of resources, but the latter is somewhat conceptually simpler.
  Having a separate instance may reduce the advantages listed above.
* The CCS image writer code would need to be enhanced to add robustness and fault tolerance and to interface with the OODS, DBB, and Prompt Processing.
  The mechanisms used by the current DM Archiver should serve as a reference, but they would have to be ported to the Java environment of the CCS.
* The CCS image writer code needs to be able to interface with the Catch-Up Archiver.
  As mentioned above, the ability to retrieve images by name from a catalog listting is planned, but it may need to be implemented sooner and as part of a new CSC.
* Locating the entire OODS or DBB at the Summit is considered impossible at LSSTCam scale.
  Either the images would have to be copied from the Camera Diagnostic Cluster to the Base for ingest into those systems or direct ingest from the Summit to the Base would need to be arranged (skipped in the event of network outage).
  One possibility is to have the CCS image writer trigger a network copy to the Base upon successful image capture and then use the current "hand-off" mechanism to the OODS and DBB.
  This may require extending the CCS image writer to send messages or write to a shared database.
* Prompt Processing should be fed directly by an international network copy from the Camera Diagnostic Cluster, rather than having an extra hop through the Base, in order to minimize latency.

The first steps in a transition to this design would be:

* Have the image writer get metadata from the Header Service.
  This is already planned, but it would be critical to get this in place ASAP.
* After successful image capture, copy the image (with metadata header) to a hand-off machine.
  Send any messages or update any databases required to use the current OODS/DBB ingest code.
  At this point, minimal functionality would be available for LATISS and test stands, including ComCam.
* Implement the current Archiver telemetry and "successful ingest" events.
* Upgrade the CCS image writer with Archiver-based robustness.

While Tony Johnson (the prime CCS author) is quite busy with LATISS commissioning, ComCam testing in Tucson, and LSSTCam integration and testing at SLAC, at least Steve Pietrowicz from NCSA could help with the Java-based aspects of this transition.

Header Service
--------------

Another possible simplification is to integrate the Header Service with the CCS image writer code.
This has potential difficulties:

* There will be a separate instance of the CCS image writer for the AOS.
  It may be difficult to keep these instances in sync or to keep multiple metadata objects separate.
* Porting the current SAL-heavy Python code to Java may not be easy.

Nevertheless, this should be considered down the road, again because having the CCS perform this function can help ensure that it happens for every image and moves the metadata capture point close to the authoritative source for most of it.

.. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
   :style: lsst_aa
