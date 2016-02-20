###################
Примеры на Solidity
###################

.. index:: voting, ballot

.. _voting:

***********
Голосование
***********

Следующий контракт довольно сложен, но демонстрирует многие возможности Solidity. Он реализует контракт голосования. Конечно, главные проблемы электронного голосования - это как назначить права голосования правильным участникам и как предтвратить манипулирование. Мы не решим здесь все проблемы, но, по крайней мере, мы покажем, как можно выполнять делегированное голсование, чтобы подсчет голосов велся одновременно **автоматически и полностью прозрачно**.

Идея в том, чтобы создать один контракт на голосование, предоставляя короткое имя для каждого варианта. Затем создатель голосования, который играет роль chairperson, предоставит право голосовать каждому адресу по отдельности.

Люди, контролирующие адреса, затем могут выбрать или голосовать самим или делегировать свой голос человеку, которому они доверяют.

В конце срока голосования функция `winningProposal()` возвращает предложение, набравшее наибольшее количество голосов.

.. Gist: 618560d3f740204d46a5

::

    /// @title Голосование с делегированием.
    contract Ballot
    {
        // Объявление нового сложного типа, который будет позже
        // использован для хранения переменных.
        // Он представляет одного избирателя.
        struct Voter
        {
            uint weight; // вес, накопленный в результате делегирования
            bool voted;  // если true, этот человек уже проголосовал
            address delegate; // человек, которому этот избиратель делегировал голос
            uint vote;   // индекс предложения, за которое был отдан голос
        }
        // Это тип одного предложения.
        struct Proposal
        {
            bytes32 name;   // краткое имя (до 32 байтов)
            uint voteCount; // количество накопленных голосов
        }

        address public chairperson;
        // Объявление переменной состояния, которая
        // хранит структуру `Voter` для каждого возможного адреса.
        mapping(address => Voter) public voters;
        // Динамический массив структур `Proposal`.
        Proposal[] public proposals;

        /// Создание нового ballot для выбора одного из `proposalNames`.
        function Ballot(bytes32[] proposalNames)
        {
            chairperson = msg.sender;
            voters[chairperson].weight = 1;
            // Для каждого из предоставленных названий предложений
            // мы создаем новый объект предложения и добавляем его
            // а концу массива.
            for (uint i = 0; i < proposalNames.length; i++)
                // `Proposal({...})` создает временный объект
                // Proposal, а `proposal.push(...)`
                // присоединяет его к концу `proposals`.
                proposals.push(Proposal({
                    name: proposalNames[i],
                    voteCount: 0
                }));
        }

        // Предоставляем `voter` право голосовать в этом голосовании.
        // Функцию может вызывать только `chairperson`.
        function giveRightToVote(address voter)
        {
            if (msg.sender != chairperson || voters[voter].voted)
                // `throw` завершает и обращает все изменения состояния
                // и балансов эфира. Часто разумно использовать
                // такой прием, если функции
                // вызываются неправильно. Но учтите, что это также потребит
                // весь предоставленный газ.
                throw;
            voters[voter].weight = 1;
        }

        /// Делегирование голоса избирателю `to`.
        function delegate(address to)
        {
            // назначение ссылки
            Voter sender = voters[msg.sender];
            if (sender.voted)
                throw;
            // Перенаправление делегирования, если
            // `to` также делегируется.
            while (voters[to].delegate != address(0) &&
                   voters[to].delegate != msg.sender)
                to = voters[to].delegate;
            // В делегировании обнаружен цикл, что не допускается.
            if (to == msg.sender)
                throw;
            // Поскольку `sender` является ссылкой, этот код
            // изменяет `voters[msg.sender].voted`
            sender.voted = true;
            sender.delegate = to;
            Voter delegate = voters[to];
            if (delegate.voted)
                // Если делегат уже проголосовал,
                // значение непосредственно добавляется к количеству голосов 
                proposals[delegate.vote].voteCount += sender.weight;
            else
                // Если делегат еще не голосовал,
                // добавляется к его весу.
                delegate.weight += sender.weight;
        }

        /// Отдать голос (включая голоса, делегированные вам)
        /// за предложение `proposals[proposal].name`.
        function vote(uint proposal)
        {
            Voter sender = voters[msg.sender];
            if (sender.voted) throw;
            sender.voted = true;
            sender.vote = proposal;
            // Если `proposal` находится за пределами массива,
            // этот код автоматически throw и обратит все
            // изменения.
            proposals[proposal].voteCount += sender.weight;
        }

        /// @dev Вычисляет победившее предложение, учитывая все
        /// предыдущие голоса.
        function winningProposal() constant
                returns (uint winningProposal)
        {
            uint winningVoteCount = 0;
            for (uint p = 0; p < proposals.length; p++)
            {
                if (proposals[p].voteCount > winningVoteCount)
                {
                    winningVoteCount = proposals[p].voteCount;
                    winningProposal = p;
                }
            }
        }
    }

Возможные улучшения
===================

В настоящее время требуется выполнить много транзакций, чтобы назначить право голоса всем участникам. Можете ли вы придумать способ лучше?

.. index:: auction;blind, auction;open, blind auction, open auction

***************
Аукцион вслепую
***************

В этом разделе мы покажем, насколько легко в Эфириуме создать контракт полностью слепого аукциона. Мы начнем с открытого аукциона, в котором каждый может видеть ставки, и затем расширим этот контракт в слепой аукцион, в котором невозможно видеть фактическую ставку до завершения периода ставок.

Простой открытый аукцион
========================

The general idea of the following simple auction contract
is that everyone can send their bids during
a bidding period. The bids already include sending
money / ether in order to bind the bidders to their
bid. If the highest bid is raised, the previously
highest bidder gets her money back.
After the end of the bidding period, the
contract has to be called manually for the
beneficiary to receive his money - contracts cannot
activate themselves.

.. {% include open_link gist="48cd2b65ff83bd04f7af" %}

::

    contract SimpleAuction {
        // Parameters of the auction. Times are either
        // absolute unix timestamps (seconds since 1970-01-01)
        // ore time periods in seconds.
        address public beneficiary;
        uint public auctionStart;
        uint public biddingTime;

        // Current state of the auction.
        address public highestBidder;
        uint public highestBid;

        // Set to true at the end, disallows any change
        bool ended;

        // Events that will be fired on changes.
        event HighestBidIncreased(address bidder, uint amount);
        event AuctionEnded(address winner, uint amount);

        // The following is a so-called natspec comment,
        // recognizable by the three slashes.
        // It will be shown when the user is asked to
        // confirm a transaction.

        /// Create a simple auction with `_biddingTime`
        /// seconds bidding time on behalf of the
        /// beneficiary address `_beneficiary`.
        function SimpleAuction(uint _biddingTime,
                               address _beneficiary) {
            beneficiary = _beneficiary;
            auctionStart = now;
            biddingTime = _biddingTime;
        }

        /// Bid on the auction with the value sent
        /// together with this transaction.
        /// The value will only be refunded if the
        /// auction is not won.
        function bid() {
            // No arguments are necessary, all
            // information is already part of
            // the transaction.
            if (now > auctionStart + biddingTime)
                // Revert the call if the bidding
                // period is over.
                throw;
            if (msg.value <= highestBid)
                // If the bid is not higher, send the
                // money back.
                throw;
            if (highestBidder != 0)
                highestBidder.send(highestBid);
            highestBidder = msg.sender;
            highestBid = msg.value;
            HighestBidIncreased(msg.sender, msg.value);
        }

        /// End the auction and send the highest bid
        /// to the beneficiary.
        function auctionEnd() {
            if (now <= auctionStart + biddingTime)
                throw; // auction did not yet end
            if (ended)
                throw; // this function has already been called
            AuctionEnded(highestBidder, highestBid);
            // We send all the money we have, because some
            // of the refunds might have failed.
            beneficiary.send(this.balance);
            ended = true;
        }

        function () {
            // This function gets executed if a
            // transaction with invalid data is sent to
            // the contract or just ether without data.
            // We revert the send so that no-one
            // accidentally loses money when using the
            // contract.
            throw;
        }
    }

Blind Auction
================

The previous open auction is extended to a blind auction
in the following. The advantage of a blind auction is
that there is no time pressure towards the end of
the bidding period. Creating a blind auction on a
transparent computing platform might sound like a
contradiction, but cryptography comes to the rescue.

During the **bidding period**, a bidder does not
actually send her bid, but only a hashed version of it.
Since it is currently considered practically impossible
to find two (sufficiently long) values whose hash
values are equal, the bidder commits to the bid by that.
After the end of the bidding period, the bidders have
to reveal their bids: They send their values
unencrypted and the contract checks that the hash value
is the same as the one provided during the bidding period.

Another challenge is how to make the auction
**binding and blind** at the same time: The only way to
prevent the bidder from just not sending the money
after he won the auction is to make her send it
together with the bid. Since value transfers cannot
be blinded in Ethereum, anyone can see the value.

The following contract solves this problem by
accepting any value that is at least as large as
the bid. Since this can of course only be checked during
the reveal phase, some bids might be **invalid**, and
this is on purpose (it even provides an explicit
flag to place invalid bids with high value transfers):
Bidders can confuse competition by placing several
high or low invalid bids.


.. {% include open_link gist="70528429c2cd867dd1d6" %}

::

    contract BlindAuction
    {
        struct Bid
        {
            bytes32 blindedBid;
            uint deposit;
        }
        address public beneficiary;
        uint public auctionStart;
        uint public biddingEnd;
        uint public revealEnd;
        bool public ended;

        mapping(address => Bid[]) public bids;

        address public highestBidder;
        uint public highestBid;

        event AuctionEnded(address winner, uint highestBid);

        /// Modifiers are a convenient way to validate inputs to
        /// functions. `onlyBefore` is applied to `bid` below:
        /// The new function body is the modifier's body where
        /// `_` is replaced by the old function body.
        modifier onlyBefore(uint _time) { if (now >= _time) throw; _ }
        modifier onlyAfter(uint _time) { if (now <= _time) throw; _ }

        function BlindAuction(uint _biddingTime,
                                uint _revealTime,
                                address _beneficiary)
        {
            beneficiary = _beneficiary;
            auctionStart = now;
            biddingEnd = now + _biddingTime;
            revealEnd = biddingEnd + _revealTime;
        }

        /// Place a blinded bid with `_blindedBid` = sha3(value,
        /// fake, secret).
        /// The sent ether is only refunded if the bid is correctly
        /// revealed in the revealing phase. The bid is valid if the
        /// ether sent together with the bid is at least "value" and
        /// "fake" is not true. Setting "fake" to true and sending
        /// not the exact amount are ways to hide the real bid but
        /// still make the required deposit. The same address can
        /// place multiple bids.
        function bid(bytes32 _blindedBid)
            onlyBefore(biddingEnd)
        {
            bids[msg.sender].push(Bid({
                blindedBid: _blindedBid,
                deposit: msg.value
            }));
        }

        /// Reveal your blinded bids. You will get a refund for all
        /// correctly blinded invalid bids and for all bids except for
        /// the totally highest.
        function reveal(uint[] _values, bool[] _fake,
                        bytes32[] _secret)
            onlyAfter(biddingEnd)
            onlyBefore(revealEnd)
        {
            uint length = bids[msg.sender].length;
            if (_values.length != length || _fake.length != length ||
                        _secret.length != length)
                throw;
            uint refund;
            for (uint i = 0; i < length; i++)
            {
                var bid = bids[msg.sender][i];
                var (value, fake, secret) =
                        (_values[i], _fake[i], _secret[i]);
                if (bid.blindedBid != sha3(value, fake, secret))
                    // Bid was not actually revealed.
                    // Do not refund deposit.
                    continue;
                refund += bid.deposit;
                if (!fake && bid.deposit >= value)
                    if (placeBid(msg.sender, value))
                        refund -= value;
                // Make it impossible for the sender to re-claim
                // the same deposit.
                bid.blindedBid = 0;
            }
            msg.sender.send(refund);
        }

        // This is an "internal" function which means that it
        // can only be called from the contract itself (or from
        // derived contracts).
        function placeBid(address bidder, uint value) internal
                returns (bool success)
        {
            if (value <= highestBid)
                return false;
            if (highestBidder != 0)
                // Refund the previously highest bidder.
                highestBidder.send(highestBid);
            highestBid = value;
            highestBidder = bidder;
            return true;
        }

        /// End the auction and send the highest bid
        /// to the beneficiary.
        function auctionEnd()
            onlyAfter(revealEnd)
        {
            if (ended) throw;
            AuctionEnded(highestBidder, highestBid);
            // We send all the money we have, because some
            // of the refunds might have failed.
            beneficiary.send(this.balance);
            ended = true;
        }

        function () { throw; }
    }

.. index:: purchase, remote purchase, escrow

********************
Safe Remote Purchase
********************

.. {% include open_link gist="b16e8e76a423b7671e99" %}

::

    contract Purchase
    {
        uint public value;
        address public seller;
        address public buyer;
        enum State { Created, Locked, Inactive }
        State public state;
        function Purchase()
        {
            seller = msg.sender;
            value = msg.value / 2;
            if (2 * value != msg.value) throw;
        }
        modifier require(bool _condition)
        {
            if (!_condition) throw;
            _
        }
        modifier onlyBuyer()
        {
            if (msg.sender != buyer) throw;
            _
        }
        modifier onlySeller()
        {
            if (msg.sender != seller) throw;
            _
        }
        modifier inState(State _state)
        {
            if (state != _state) throw;
            _
        }
        event aborted();
        event purchaseConfirmed();
        event itemReceived();

        /// Abort the purchase and reclaim the ether.
        /// Can only be called by the seller before
        /// the contract is locked.
        function abort()
            onlySeller
            inState(State.Created)
        {
            aborted();
            seller.send(this.balance);
            state = State.Inactive;
        }
        /// Confirm the purchase as buyer.
        /// Transaction has to include `2 * value` ether.
        /// The ether will be locked until confirmReceived
        /// is called.
        function confirmPurchase()
            inState(State.Created)
            require(msg.value == 2 * value)
        {
            purchaseConfirmed();
            buyer = msg.sender;
            state = State.Locked;
        }
        /// Confirm that you (the buyer) received the item.
        /// This will release the locked ether.
        function confirmReceived()
            onlyBuyer
            inState(State.Locked)
        {
            itemReceived();
            buyer.send(value); // We ignore the return value on purpose
            seller.send(this.balance);
            state = State.Inactive;
        }
        function() { throw; }
    }

********************
Micropayment Channel
********************

To be written.
