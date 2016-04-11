# Example #

```
import org.bulatnig.smpp.net.impl.TcpConnection;
import org.bulatnig.smpp.pdu.CommandStatus;
import org.bulatnig.smpp.pdu.Npi;
import org.bulatnig.smpp.pdu.Pdu;
import org.bulatnig.smpp.pdu.Ton;
import org.bulatnig.smpp.pdu.impl.*;
import org.bulatnig.smpp.session.MessageListener;
import org.bulatnig.smpp.session.Session;
import org.bulatnig.smpp.session.impl.BasicSession;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.util.concurrent.atomic.AtomicLong;

public class SmscDeliveryReceiptTest {

    protected final Logger logger = LoggerFactory.getLogger(SmscDeliveryReceiptTest.class);

    private Session session;

    public void test() {
        try {
            // Prepare bind request
            BindTransceiver bindRequest = new BindTransceiver();
            bindRequest.setSystemId("id");
            bindRequest.setPassword("pass");
            bindRequest.setAddrTon(Ton.INTERNATIONAL);
            bindRequest.setAddrNpi(Npi.ISDN);

            // Create session and bind
            session = new BasicSession(new TcpConnection(new InetSocketAddress("host", 12345)));
            session.setMessageListener(new MessageListenerImpl());
            Pdu bindResponse = session.open(bindRequest);
            long commandStatus = bindResponse.getCommandStatus();
            if (CommandStatus.ESME_ROK != commandStatus)
                throw new IOException("Bind failed with error: " + commandStatus + ".");

            // Prepare SubmitSm request
            SubmitSm submitSm = new SubmitSm();
            submitSm.setSourceAddrNpi(Npi.ISDN);
            submitSm.setSourceAddr("1234");
            submitSm.setDestAddrTon(Ton.INTERNATIONAL);
            submitSm.setDestAddrNpi(Npi.ISDN);
            submitSm.setDestinationAddr("74952235555");
            submitSm.setRegisteredDelivery(1);
            submitSm.setShortMessage("Hello, how are you?".getBytes("US-ASCII"));

            // Set PDU sequenceNumber, which define corresponding SubmitSmResp
            submitSm.setSequenceNumber(session.nextSequenceNumber());

            // Send SubmitSm
            session.send(submitSm);

            // Wait for SMSC Delivery Receipt some time.
            Thread.sleep(60000);
        } catch (Exception e) {
            logger.error("Unknown test error.", e);
        } finally {
            if (session != null)
                session.close();
        }
    }

    private class MessageListenerImpl implements MessageListener {

        private AtomicLong sent = new AtomicLong(0);
        private AtomicLong successfullySent = new AtomicLong(0);
        private AtomicLong delivered = new AtomicLong(0);

        @Override
        public void received(Pdu pdu) {
            if (pdu instanceof DeliverSm) {
                delivered.incrementAndGet();
                DeliverSm deliverSm = (DeliverSm) pdu;
                logger.info("DeliverSm test: {}.", new String(deliverSm.getShortMessage()));
                DeliverSmResp deliverSmResp = new DeliverSmResp();
                deliverSmResp.setSequenceNumber(deliverSm.getSequenceNumber());
                try {
                    session.send(deliverSmResp);
                } catch (Exception e) {
                    logger.error("DeliverSmResp send failed.", e);
                }
                logger.info("Delivered total: {}.", delivered.get());
            } else if (pdu instanceof SubmitSmResp) {
                sent.incrementAndGet();
                if (0 == pdu.getCommandStatus())
                    successfullySent.incrementAndGet();
                logger.info("Sent total = {}, successful = {}.", sent.get(), successfullySent.get());
            } else {
                logger.info("No handler for {} found.", pdu.getClass().getName());
            }
        }
    }
}
```

Example log output

```
13:36:29.424 DEBUG - >>> 0000001f000000090000000000000001746573740070617373000000010100
13:36:29.432 DEBUG - <<< 0000001480000009000000000000000173797300
13:36:29.434 DEBUG - >>> 00000043000000040000000000000002000001313233340001013734393532323335353535000000000000010000001348656c6c6f2c20686f772061726520796f753f
13:36:29.440 DEBUG - <<< 000000248000000400000000000000023334323737363839353831323331353438353500
13:36:29.441 INFO - Sent total = 1, successful = 1.
13:36:29.740 DEBUG - <<< 000000a7000000050000000000000001000101373439353232333535353500010131323334000400000000000000007769643a33343237373638393538313233313534383535207375623a30303120646c7672643a303031207375626d697420646174653a3131303132313039333020646f6e6520646174653a3131303132313039333520737461743a44454c49565244206572723a30303020746578743a736f6d6574657874
13:36:29.740 INFO - DeliverSm test: id:3427768958123154855 sub:001 dlvrd:001 submit date:1101210930 done date:1101210935 stat:DELIVRD err:000 text:sometext.
13:36:29.741 DEBUG - >>> 0000001180000005000000000000000100
13:36:29.741 INFO - Delivered total: 1.

```