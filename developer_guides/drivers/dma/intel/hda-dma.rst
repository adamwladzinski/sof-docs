.. _intel-cavs-hda-dma-driver:

cAVS HD/A DMA Driver
####################

Initialization
**************

Internal data structures are initialized, there is no need to touch the HW at
this point.

Setting up Callback
*******************

A client (host/dai component for instance) registers a callback by calling
``dma_set_cb()`` to be notified every time next segment of data is transferred.
Size of the segments is specified by SG elements passed to
``dma_set_config()``.

Starting the Device
*******************

The device is registered in the PM platform driver, to make sure DMI L1 is
handled properly.

.. note:: It is not required when the PM call is made by the device
   driver, but when the call is moved to the systick handler, the PM
   platform driver must know whether there are any active DMA devices
   registered.

.. uml:: images/hda-start-flow.pu
   :caption: HD/A DMA Device Start Flow

Stopping the Device
*******************

The device is unregistered from the Platform PM driver.

HW reset is programmed by setting ``GEN`` to 0, DSP confirms ``GBUSY`` is 0,
otherwise exception is reported to the host.

.. uml:: images/hda-stop-flow.pu
   :caption: HD/A DMA Device Stop Flow

Transferring Data
*****************

Transmission is started on the DSP side right after the ``dma_start()`` is
called as ``GEN`` is set to 1 there. Since there are no buffer completion
interrupts available, the DSP has to poll the write/read pointer positions in
order to notify the client that transmission of next segment is finished.

Non-cyclic Mode (host input/output DMA)
=======================================

Polling should be started with a delay, when the transmission is expected
to complete successfully. The dma driver registers a callback in the default
system work queue with the appropriate timeout that is as short as possible
but long enough for the first period transfer to complete (TBD value).

.. uml:: images/hda-host-first-period.pu
   :caption: First callback notification for host playback

.. uml:: images/hda-host-next-period.pu
   :caption: Next callback notification for host playback and capture

Cyclic Mode (link input/output DMA)
===================================

On the link side, the HD/A DMA pointers are updated in real-time with a small
step. Therefore the delayed work is scheduled with the configured audio period
(1 ms usually) timeout.

.. uml:: images/hda-link-period.pu
   :caption: Callback notification for link playback and capture

Power Management
================

There is a PM call at the end of ``hda_dma_copy()`` to force
DMI L1 Exit for configured period of time, in order to make sure
the DMA does not enter L1 before the transfers are finished.
