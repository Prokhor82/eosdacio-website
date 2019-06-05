### О чём идет речь?
На днях меня спросили о eosDAC, и что же особенного в нашей работе по созданию такой "странной" вещи, которую мы называем DAC инструменты. Чем отличается это решение от централизованных решений для управления организациями и их операциями? Насколько это безопасно? Как мы смогли разработать и построить коллекцию смарт-контрактов, гарантирующую, что ни одна другая организация не сможет навредить DAC через смарт-контракты? 

Уже было несколько статей и постов в нашем блоге, обсуждающих основные принципы и управление DAC с помощью программного кода, в этой же статье сосредоточимся на смарт-контрактах, и как они работают для достижения безопасного автономного управления. Весь исходный код доступен для просмотра/использования/адаптации на Github, но не все же знакомы с C++.

Одна из главных идей во всех наших смарт-контрактах – это возможность их повторного использования другими DAC, чтобы можно было без проблем скопировать и использовать наш исходный код. С самого начала это стало для нас основным убеждением и оправданием того, почему некоторые решения стали более сложными, чем могли бы быть, если бы мы разрабатывали решение исключительно для нас. Будущая цель - сделать их еще более удобными, чтобы другие DAC могли использовать наши смарт-контракты и настраивать их параметры для своих собственных целей без необходимости переписывать исходный программный код. Таким образом, они получат все обновления одновременно вместе с командой eosDAC при добавлении новых функций и исправлении возможных ошибок. 

Код контрактов состоит из контрактов для токенов (`eosdactokens`), для выборов и управления хранителями (`dacustodian`), для подачи и управлению рабочими предложениями, а также для оплаты и координации отдельного депозитного счета (`dacescrow`), чтобы надежно сохранить средства рабочего предложения. Это необходимо, как для защиты самого DAC, так и работника от возможной потери оплаты за выполненную работу.

## EOSDACTOKENS

С этого контракта всё и началось. Токены EOSDAC хранятся в этом смарт-контракте. Все началось с копирования `eosio.token` - главного контракта для токенов, который используется и для токена EOS, а также является отправной точкой для всех токенов, работающих в сетях EOS. Мы добавили некоторые функциональные возможности к этому контракту, чтобы соответствовать нашим потребностям для запуска DAC и первоначального эирдропа, и включают в себя следующее:

### Возможность создания токенов в заблокированном состоянии
Основная цель заключалась в том, чтобы во время первоначального дропа держателям оригинальных токенов на базе Ethereum в июне 2018 года, мы бы смогли провести тщательную проверку балансов аккаунтов получателей и убедились, что они соответствуют ожидаемым остаткам при снимке сети Ethereum и прежде чем пользователи смогут начать торговать своими токенами. Мы очень серьезно отнеслись к тестированию :)

EOSDAC был одним первым токеном, который сделал эирдроп в сети EOS, так что многоt было неизвестно. Как только дроп был завершен и протестирован, мы смогли разблокировать токены и разрешить их передачу. Важно отметить, что нет возможности повторно заблокировать токен после того, как его разблокировали, так как эта функция могла бы позволить создателям токенов манипулировать ценой посредством централизованного контроля ликвидности токенов.

### Условия принятия участников
Согласие с условиями использования в рамках DAC рассматривалось нами как фундаментальная ключевая особенность смарт-контрактов DAC, и поэтому имело смысл включить эту функциональность в контракт токенов. Пользователь становится зарегистрированным участником сообщества после подтверждения и выполнения действия `memberreg` по контракту. 

Для выполнения этого действия пользователь должен будет согласиться с условиями, и тем самым подтвердит контрольную сумму с известной контрольной суммой последних условий, прежде чем стать зарегистрированным участником. Эта логика контрольной суммы абстрагируется от конечного пользователя через графический интерфейс, и предоставляет криптографическое подтверждение того, что пользователь согласился с определенным набором терминов и добавляет своё согласие для взаимодействия с кодом контракта. Пока выполняются эти условия и совпадает контрольная сумма известных условий действие `memberreg` успешно выполняется. Хотя это может показаться слишком сложным способом подтверждения согласия с терминами и условиями, мы специально выбрали такой подход, потому что это все равно что сказать «я согласен *с этими* условиями» вместо того, чтобы спрашивать пользователя «Согласны ли вы с последними положениями и условиями» с ответом пользователя «да» или «нет». Мы считаем, что последнее будет менее надежным, так как пользователь может случайно принять неправильные условия, которые уже устарели в контракте. 

В интересах экономии использования RAM для всех конечных пользователей в контракте хранится только ссылка на хэш принятых условий, хранящийся в контракте, а не копия принятого хэша для каждого пользователя. Это означает, что каждому пользователю нужно хранить только версию принятых терминов (8 байт), а не весь хэш (32 байта) , и поскольку использование RAM в сети дорого, любая небольшая экономия нам помогает. 

Состояние принятия последних условий каждым участником неоднократно упоминаются в смарт-контрактах EOSDAC, чтобы гарантировать, что пользователь не сможет выполнять действия в DAC без согласия с последними условиями, в том числе и в будущем при обновлении таких условий.

## DACCUSTODIAN

Этот контракт управляет всеми ставками, выдвижением, голосованием, подсчетом голосов и назначением хранителей по завершении каждого периода выборов. Конечным результатом действий этого контракта является управление разрешениями на управляющей учетной записи DAC (`dacauthority`), настраиваемыми с помощью конфигураций этого контракта. Управляющая учетная запись имеет заранее определенные разрешения на все действия в DAC, включая изменение кода и перевод средств. Эти разрешенные действия управляются различными транзакциями с мультиподписью, используя встроенные сложные инструменты управления разрешениями, доступные в программном обеспечении протокола EOSIO. Параметры избирательного процесса могут быть изменены с помощью конфигурации, установленной в контракте при помощи действия `updateconfig`. Функциональность этого контракта можно разбить на следующие разделы:

### Cтекирование
Перед тем, как выдвинуть себя в качестве кандидата, участник должен сначала перенести определенную сумму EOSDAC, используя действие `transfer` по контракту токена на этот счет. Поле Memo должно быть одинаковым с именем учетной записи этого контракта (например "daccustodian") для того, чтобы перенос был признан как транзакция по стекингу, а требуемая сумма настраивается полем `lockupasset` в объекте config. После установки ставки в контракте будет указана сумма ожидающей ставки для этого кандидата, готового к выдвижению.

### Выдвижение кандидатов - `nominatecand`:
Чтобы стать избранным хранителем DAC, зарегистрированный участник должен сначала выдвинуть себя в качестве кандидата, используя действие`nominatecand`. Для этого потребуется указать имя выдвигаемой учетной записи и запрашиваемую сумму оплаты в EOS. Это действие проверяет, что пользователь является зарегистрированным участником и стекировал достаточное количество токенов EOSDAC. Также контракт проверяет, что запрашиваемая сумма оплаты не превышает допустимое значение `requested_pay_max` из конфигурации, и если все эти условия будут выполнены, добавит пользователя в качестве кандидата для голосования.

### Голосование - `votecust`:
Голосование осуществляется действием `votecust` и потребует учетные запись избирателя и наличие списка кандидатов, за которых вы хотите проголосовать. Голосование проводится по следующим правилам: 

* Голосование требует действительного членства и согласия с текущими условиями участия.
* Вес каждого голоса определяется балансом EOSDAC, который избиратель имеет на своем счете. (В настоящее время для голосования не требуется никаких ставок)
* После того, как голоса избирателя будут размещены, это голосование остается активным до тех пор, пока избиратель не изменит свои голоса. Если баланс аккаунта упадёт до 0, а затем снова станет положительным или один из кандидатов уходит в отставку, а затем возвращается, голосование за него по-прежнему будет активным и применимым при подсчете голосов.
* Все кандидаты, за которых проголосовал избиратель, получают одинаковый вес голоса на основе баланса EOSDAC.
* Существует максимальное количество кандидатов, за которых избиратель может голосовать, определяемое настройкой конфигурации.
* Чтобы удалить голосование, избирателю нужно будет проголосовать с пустым списком кандидатов.
* Голосование и передача токенов для голосования будет иметь прямое и непосредственное влияние на вес каждого активного голосования, но единственный раз, когда вес голосов будет иметь значение - это блок, когда вызывается действие `newperiod`. (Объяснено ниже)

### Новый период выборов - `newperiod`:
Для каждого действительного кандидата вес голоса будет непрерывно отслеживаться на основе изменений, вызванных действием `votecust` или действием `transfer` в контракте токена, если есть активное голосование за учетные записи `from` или `to` в действии передачи. При непрерывном отслеживании этих значений фактические выборы (во время `newperiod`) должны только сделать снимок кандидатов, выбранных большинством голосов в этот момент. Количество кандидатов настраивается с помощью поля `numelected` в конфигурации. Для успешного выполнения действия `newperiod` потребуются другие проверки, в том числе:

* Ensure the initial number of voters exceeds the `initial_vote_quorum_percent` config. 
* Ensure on subsequent elections the number of voters exceeds `vote_quorum_percent` config.
* Ensure the time period between calls to `newperiod` exceeds the `periodlength` time delay.

If all these checks succeed the following events are performed:

* Custodian pay is distributed as per the median amount of the current custodians `requestedpay` amounts.
* The new custodians are allocated based the ranking weight of their accumulated votes for the next period.
* The permissions for operating the DAC accounts are updated to include the newly appointed custodians.
* The current time is saved to use as a time reference for future calls the `newperiod` to ensure it's not called too early for the next period.
* For each candidate elected as a custodian their staked tokens will become locked up at this time to show they have some skin in the game. They will be able unstake their tokens once they are no longer a elected after a lockup period determined by `lockup_release_time_delay` from the config.

### Other actions:
In addition to the happy path as explained above there are other actions supported in this contract including:

* Withdraw candidate (`withdrawcand`): This would be called by an existing candidate, including one that is currently an elected custodian when they would like to remove themselves from the next period of elections. Their tokens will remain staked and if the tokens are also locked up they will remain locked until the release time expires. A good use case for this action may be a custodian going on holidays with the intention of returning soon (that's the reason for keeping the unstake functionality separate). Otherwise this action could be used for any other reason for a candidate voluntarily withdrawing from participating in the DAC operations.
* fire a candidate (`firecand`): This would be called by the currently elected custodians via the multi-sig auth account to remove a misbehaving candidate. This action has the option to lockup the candidates staked tokens if they are not already locked up.
* fire a custodian (`firecust`): Similar to the `firecand` this action will remove a currently elected custodian as actioned by the other custodians. This will remove the custodian from elected custodian group, remove them as a potential candidate for future elections, replace the custodian with the next highest voted candidate and finally update the account permissions to reflect the new set of elected custodians.
* resign a custodian (`resigncust`): This action must be run by an active custodian who would like to resign from being an active custodian. This would remove them from an eligible candidate and replace the custodian with the next highest voted candidate.
* Update requested pay (`updatereqpay`): A candidate may update their requested pay as a custodian. This will not take effect until the next period to prevent mid period changes to custodian pay.
* Claim pay (`claimpay`): This is the action called by an active custodian in order to receive the pay amount that is due to to be paid to them. Due to the legacy legal system we live in the payment is made via a service company for legal reasons beyond the scope of this document. From the contract's perspective this is the action to pay a custodian for their services to the DAC as a custodian and after this action has been performed the custodian is considered paid.
* Update config (`updateconfig`): There are several configurable options on the contract code to allow it to be customised for use without needing to recompile and deploy the code. This future plan is that this will allow different DACs to operate using the same deployed code but with their own custom configurations. The changes are made by setting a new config object via this action with the following options:

	* `lockupasset`: The amount of EOSDAC token that are locked up by each candidate applying for election.
	* `maxvotes` : The maximum number of votes that each member can make for a candidate. default of 5.
	* `numelected` : Number of custodians to be elected for each election count.
	* `periodlength` : Length of a period in seconds. Used to prevent early elections from being called on the `newperiod` action. default 7 days.
	* `authaccount` : controlling account to have it's permissions set with the elected custodians.
	* `tokenholder` : The contract that holds the funds for the DAC. This is used as the source for custodian pay.
	* `serviceprovider` :  The contract that will act as the service provider account for the DAC. This is used as the deliverer of pay to custodians and workers on worker proposals.
	* `should_pay_via_service_provider` : If set to true the contract will direct all payments via the service provider rather than paying directly.
	* `initial_vote_quorum_percent` :  Amount of token value in votes required to trigger the initial set of custodians
	* `vote_quorum_percent` : Amount of token value in votes required to allow a new set of custodians to be set after the initial threshold has been achieved - election period 2 and onwards.

		The required number of custodians to approve different levels of authenticated actions on the DAC smart contracts:	

	* `auth_threshold_high`
	* `auth_threshold_mid`
	* `auth_threshold_low`
	* `lockup_release_time_delay` : The time before locked up stake can be released back to the candidate using the unstake action
	* `requested_pay_max` : The maximum amount of pay a custodian can request for payment.

## DACPROPOSALS

This contract is responsible for managing the worker proposals related to the DAC. It is once again built for configurability rather than just to suit our immediate needs in EosDAC. 

The general idea is that potential worker will have a piece of work they would like to propose to add value to the DAC in exchange for being paid in an amount of EOS based tokens. The proposal would be voted for approval for commencement and then completion by the current custodians and these actions would trigger payments to the proposer. 


### Create proposal `createprop` 
A proposing worker would create a proposal and submit it to the blockchain for the review and voting by the current DAC custodians. To proposal would need to include the following:

* `title` (String): to identify the proposal
* `summary` (String): a brief summary of the the purpose of the worker proposal
* `arbitrator` (EOS Account name): the account name of an independent arbitrator who may be called upon to satisfy disputes in the completion of a worker proposal.
* `pay_amount` (EOSAsset): an amount of EOS based tokens requested as the pay amount for the worker proposal.
* `content_hash` (ChecksumHash): a content hash to ensure details of a proposal stored off-chain are not modified after a proposal has been agreed to. This allows for much more extensive detail that would not be stored on chain while still maintaining data integrity.

For each proposal minimal content data is required to be stored in the contract state and is instead only passed through for data integrity via transaction logs. Only the account and payment data is stored for utilisation in later actions within this contract.

### Voting for a proposal `voteprop`
Once a proposal has been created it would be in a state waiting for the current custodians to vote either 'proposal_approve' or 'proposal_deny' for a proposal with the required number of votes and number of 'yes' votes to be configurable in the contract. At this time there may be refinements to the proposal with canceling `cancel` of existing proposals and resubmitting changes based on feedback from the custodians until the proposals get to ready and positively-voted-for position.

### Start work on an accepted proposal `startwork`
It there have been sufficient positive votes for a proposal the proposer will be able to call this action to confirm that they will agree to work on the agreed proposal with the agreed terms, payment etc. At this point the agreed payment amount is transferred into an escrow account to ensure the funds can and will be paid to the proposer when the work is complete as approved by the custodians or the agreed arbitrator for the proposal.

The locking of funds in an escrow account is a crucial step to protect the worker from potentially malicious custodians who could reverse an earlier proposal acceptance because they have a trusted arbitrator on the proposal to also be able to release the funds.

### Signal completion of work `completework`
After a worker has completed their work they would signal to the custodians that work is believed to be complete and the custodians would need to assess the work before approving via another vote using `voteprop` but this time with a different choice of vote values to indicate `claim_approve` or `claim_deny`.

### Claim payment for completed work `claim`
If there have been sufficient positive votes for the completed work from the custodians based on the current configurations then the proposer can call the action which will trigger the transfer of token payments to the worker. In practice for EosDAC the funds are sent to a service company account but effectively they are paid directly to the worker for their services.

### Contract configuration `updateconfig`
Various options are configurable for this contract including:

* `service_account` (Eos account name) 
* `proposal_threshold` number of required votes to participate in voting for a proposal.
* `proposal_approval_threshold_percent` required percentage of positive votes to approve a proposal
* `claim_threshold` number of required votes to participate in voting for completing a proposal.
* `claim_approval_threshold_percent` required percentage of positive votes to approve a proposal claim.
* `escrow_expiry` the expiry time set on the created escrow transaction (number of seconds). This has a default value of 30 days.

## DACESCROW
This contract is responsible for holding the secured funds for worker proposals until the elected custodians or an agreed arbitrator releases the funds to the receiver or the escrow time limit expires which would allow returning of funds to the sender. The intention is that this contract would be locked to even prevent code modification from malicious custodians. This would be achieved via deleting the owner and active keys on the account after the contract code has been set. This desire for immutability is another reason for this contract to be separate and simple from the other code contracts in the code. The hope is that this code never needs to be modified. In the unfortunate (and hopefully very unlikely) scenario that this code does need to be modified the custodians would need to get the required agreement from the current block producers to reset the owner key for this account.

### Initialise an escrow transaction - `init`
An escrow transaction must be initialised specifying the all the required fields including the sender, intended receiver, expiry time, arbitrator, memo for the eventual transfer action. There is also an optional external key which can be used as a cross-contract reference key rather than only relying on the internal auto-incrementing key which would otherwise lead to key collisions in time.

### Transfer funds for an escrow - `transfer`
Funds for an escrow would need to be transferred to escrow contract using the usual `transfer` action as seen and replicated by most EOS based token contracts. This contract's code relies on the built in notifications that the transfer action directs to both the sender and receiver of accounts of the transfer. When a transfer notification is received by the escrow contract the `transfer` action implementation will verify the sender has an empty escrow record and assigns the amount transferred into that escrow record for later processing by the other actions in the escrow contract code. An initialised escrow record may be cancelled with the `cancel` action provided it is called before the transfer action has populated the escrow.

### Approval or un-approval of an escrow - `approve` and `unapprove`
Once an escrow has been initialised and populated with a transfer action the next step would be to approve the escrow either by the sender, the nominated arbitrator or receiver. Two approvals are required from any of these three accounts to allow the `claim` action to be performed. `unapprove` may be subsequently called to remove an existing approval by the relevant actor. 

### Claim an approved escrow payment - `claim`
The claim action can only be executed by the intended receiver for an escrow payment and will only succeed with the correct approval state for the escrow record. At this point the escrowed amount will be transferred to the nominated service company account so that payments can be processed to the intended receiver. 

_Note: The service company step is not included for technical reasons but is a legal requirement to have sufficient interfacing with the traditional legal world. As much as we would like to perform all actions in the safety of cryptographically secured smart contract environment the world is not ready :( ._

### Refund after expiry - `refund`
While the funds in the escrow account must be locked up for a certain duration they must also be available after expiry time has passed if there has not been sufficient approval from the sender or the arbitrator otherwise there could be funds locked in the account permanently. The `refund` action provide this mechanism and can only be called by the sender if the expiry time has passed. Then the escrowed amount will be transferred back the sender and the escrow record will be removed to prevent a double refund scenario.  

[Back]({% translate_link tools %})